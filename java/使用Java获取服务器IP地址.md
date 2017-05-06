使用Java获取服务器IP地址

```
public class NetworkInterfaceUtil {
    private static List<NetworkInterface> getInterfaces() throws SocketException {
        return Collections.list(NetworkInterface.getNetworkInterfaces());
    }

    private static List<NetworkInterface> getNonLoopBackInterfaces() throws SocketException {
        return getInterfaces().stream()
                              .filter(i -> Unchecked.supplier(() -> !i.isLoopback()).get())
                              .collect(toList());
    }

    public static List<String> getIp4Addresses() throws SocketException {
        final List<NetworkInterface> is = getNonLoopBackInterfaces();
        return is.stream().flatMap(i -> {
            final Enumeration<InetAddress> addresses = i.getInetAddresses();
            final Builder<String> builder = Stream.builder();
            while (addresses.hasMoreElements()) {
                final InetAddress ip = addresses.nextElement();
                if (!ip.isLoopbackAddress()) {
                    if (ip.getHostAddress().equalsIgnoreCase("127.0.0.1")) {
                        continue;
                    }
                    if (ip instanceof Inet6Address) {
                        continue;
                    }
                    if (ip instanceof Inet4Address) {
                        builder.add(ip.getHostAddress());
                    }
                }
            }
            return builder.build();
        }).collect(toList());
    }
}
```

---
write on 2017-1-4