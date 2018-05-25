server = server.start()

transportServer.start()

NettyServer的start方法，该start方法是netty server bootstrap的过程，每个链接上来的客户端会调用childHandler逻辑。

里边调用了NettyServerTransport的start方法：
  public void start(ServerTransportListener listener) {
    Preconditions.checkState(this.listener == null, "Handler already registered");
    this.listener = listener;

    // Create the Netty handler for the pipeline.
    final NettyServerHandler grpcHandler = createHandler(listener);
    NettyHandlerSettings.setAutoWindow(grpcHandler);

    // Notify when the channel closes.
    channel.closeFuture().addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
        notifyTerminated(grpcHandler.connectionError());
      }
    });

    ChannelHandler negotiationHandler = protocolNegotiator.newHandler(grpcHandler);
    channel.pipeline().addLast(negotiationHandler);
  }
里边主要逻辑是由grpcHandler负责，这个需要重点看下。

