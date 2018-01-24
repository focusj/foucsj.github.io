1. 所有的客户端请求都会封装成ClientCall。ClientCall由Channel创建，channel.newCall()。newCall是ClientCallImpl的实例，
3. ClientCall在执行的时候会出发创建底层channel的动作，在grpc中叫sub-channel。因为grpc是基于http2协议，sub-channel是可复用的。所以客户端连续发起请求，是不会重复创建channel的。

4. ManagedChannel: 封装了底层Channel，并管理channel的生命周期。创建ManagedChannel的同时会创建TransportFactory， 默认会返回NettyTransportFactory实例。它负责创建NettyClientTransport。NettyClientTransport的start方法负责开启一个channel。Http2FrameReader、Http2FrameWriter的创建时伴随NettyClientHandler的创建一并创建的。

ClientCallImpl.start()

ClientTransport transport = clientTransportProvider.get(
          new PickSubchannelArgsImpl(method, headers, callOptions));
