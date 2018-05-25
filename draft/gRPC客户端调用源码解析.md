首先调用ClientCallImpl的start执行，start方法内执行clientTransportProvider.get方法。
第一次执行的时候触发执行exitIdleMode()方法。
exitIdleMode方法里会创建LoadBalancer，NameResolver，NameResolverListenerImpl和LBHelper（LBHelper提供LoadBalancer的具体方法）。并调用LoadBalancer.start启动LoadBalancer。

NameResolver start方法中调动resolve方法，查找域名。resolve的执行过程是异步的，通过listener.onAddress把解析到的地址信息传递出去。

onAddress会把地址传给lb，balancer.handleResolvedAddressGroups。

balancer的handleResolvedAddressGroups会创建subchannel，并且告知lbhelper更新状态。（subchannel是干啥的？）

lbHelper的helper.updateBalancingState方法会调用Transport的reporess方法，把堆积的stream重新执行。执行过程中通过picker.pickSubChannel从lb选择链接。

### SubChannel：
SubChannel代表了一个逻辑上的链接（或者一些等价的socket address即EquivalentAddressGroup），最多持有一个链接，如果被用来发送rpc时没有活跃的transport则通过requestConnection()创建一个链接。否则不去主动创建链接。

createSubchannel是由Helper负责的。


public void handleResolvedAddressGroups(
    List<EquivalentAddressGroup> servers, Attributes attributes) {
    // NameResolve拿到address之后，由NameResolverListener触发lb调用handleResolvedAddressGroups方法。
    EquivalentAddressGroup newEag = flattenEquivalentAddressGroup(servers);
    if (subchannel == null) {
        // helper创建subChannel
        subchannel = helper.createSubchannel(newEag, Attributes.EMPTY);

        // The channel state does not get updated when doing name resolving today, so for the moment
        // let LB report CONNECTION and call subchannel.requestConnection() immediately.
        helper.updateBalancingState(CONNECTING, new Picker(PickResult.withSubchannel(subchannel)));
        subchannel.requestConnection();
    } else {
        helper.updateSubchannelAddresses(subchannel, newEag);
    }
}

接着看updateBalancingState，delayedTransport.reprocess会拿到实际的Transport，把pendingStream发出去。
@Override
    public void updateBalancingState(
        final ConnectivityState newState, final SubchannelPicker newPicker) {
      checkNotNull(newState, "newState");
      checkNotNull(newPicker, "newPicker");

      runSerialized(
          new Runnable() {
            @Override
            public void run() {
              if (LbHelperImpl.this != lbHelper) {
                return;
              }
              subchannelPicker = newPicker;
              delayedTransport.reprocess(newPicker);
              // It's not appropriate to report SHUTDOWN state from lb.
              // Ignore the case of newState == SHUTDOWN for now.
              if (newState != SHUTDOWN) {
                channelStateManager.gotoState(newState);
              }
            }
          });
    }



public void requestConnection() {
    subchannel.obtainActiveTransport();
}

ClientTransport obtainActiveTransport() {
    ClientTransport savedTransport = activeTransport;
    if (savedTransport != null) {
      return savedTransport;
    }
    try {
      synchronized (lock) {
        savedTransport = activeTransport;
        // Check again, since it could have changed before acquiring the lock
        if (savedTransport != null) {
          return savedTransport;
        }
        if (state.getState() == IDLE) {
          gotoNonErrorState(CONNECTING);
          startNewTransport();
        }
      }
    } finally {
      channelExecutor.drain();
    }
    return null;
  }

如果没有activeTransport，则startNewTransport创建新的transport。

  private void startNewTransport() {
    Preconditions.checkState(reconnectTask == null, "Should have no reconnectTask scheduled");

    if (addressIndex == 0) {
      connectingTimer.reset().start();
    }
    List<SocketAddress> addrs = addressGroup.getAddresses();
    final SocketAddress address = addrs.get(addressIndex);

    ProxyParameters proxy = proxyDetector.proxyFor(address);
    ConnectionClientTransport transport =
        transportFactory.newClientTransport(address, authority, userAgent, proxy);
    if (log.isLoggable(Level.FINE)) {
      log.log(Level.FINE, "[{0}] Created {1} for {2}",
          new Object[] {logId, transport.getLogId(), address});
    }
    pendingTransport = transport;
    transports.add(transport);
    Runnable runnable = transport.start(new TransportListener(transport, address));
    if (runnable != null) {
      channelExecutor.executeLater(runnable);
    }
  }
transportFactory.newClientTransport，实际有NettyTransportFactory
