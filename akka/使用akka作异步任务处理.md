同步转异步是一种常见的优化手段，最近一次在做调优时便大量使用了这种方式。通常在一个业务场景中会包含多个操作，有些操作的结果需要让用户立马知道，但有些操作则不需要。这些用户不需要等待结果的操作，我们在编程的时候便可以异步处理。这么做最直接的效果就是缩短接口响应速度，提升用户体验。

我此次优化的是下单场景。创建订单时同步操作有: 查询库存，扣款，刷新库存; 可异步的操作有: 通知风控系统，给买家发送扣款邮件和短信，通知卖家，创建一些定时任务。

最初我用的方案是Spring提供的@Async机制。这是一种很轻量的做法，只需要在可异步调用的方法上加上@Async注解即可。但是这种做法也存在两个问题: 1. 不支持类内部方法之间的调用。使用这种方式，我必须要把一些需要异步调用的方法转移到一个新类里，这点让人不爽。2. 当系统crash的时候，缓存的任务就丢了。因此，这个方案并不特别理想。

两年之前用akka做过一个社交应用的后端服务，而且消息模型天生异步，所以自然想到了用akka。但是用akka的话也有一些地方需要注意。第一，Actor是单线程顺序执行，如果任务比较多最好使用actor router。actor router管理多个actor，可以做到一定限度的并行执行。第二，使用有持久化actor，确保任务不会丢失。我会以发push提醒为例描述一下这个方案的实现细节。多数场景中发push提醒都可进行异步调用。

![classes.png](http://upload-images.jianshu.io/upload_images/78847-74c9acc26160e56d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下单逻辑都放在OrderService中，下单成功给卖家发送push提醒时，Orderservice会给NotificationActor发送一个消息。

NotificationActor有两个职责：1. 保存接收到的任务；2. 把消息转发给NotificationWorker，当Worker执行成功之后把消息删除。在最新版本的akka中可以使用[At-Least-Once Delivery](http://doc.akka.io/docs/akka/2.4/java/persistence.html#At-Least-Once_Delivery)实现这两个功能。

NotificationWorkerRouter仅仅处理发送逻辑。WorkerActor以Router方式进行部署，以实现并行处理，提高处理效率。

下边看一下具体实现细节：
```
public class NotificationActor extends UntypedPersistentActorWithAtLeastOnceDelivery {
    private final LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    private ActorRef notificationWorkers = null;
    private final String uniqueId = UUID.randomUUID().toString();

    @Autowired
    public NotificationActor(final ActorSystemManager actorSystemManager) {
        this.notificationWorkers = actorSystemManager.notificationWorkers;
    }

    @Override public String persistenceId() {
        return "journal:notification-actor:" + uniqueId;
    }

    @Override public void onReceiveRecover(final Object msg) throws Throwable {
        if (msg instanceof NotificationMessage) {
            deliverAckMessage((NotificationMessage) msg);
        }
    }

    @Override public void onReceiveCommand(final Object msg) throws Throwable {
        if (msg instanceof NotificationMessage) {
            persist(msg, m -> { deliverAckMessage((NotificationMessage) m); });
        } else if (msg instanceof Confirm) {
            Confirm confirm = (Confirm) msg;
            confirmMessage(new MsgConfirmed(confirm.deliveryId));
        } else if (msg instanceof UnconfirmedWarning) {
            UnconfirmedWarning warning = (UnconfirmedWarning) msg;
            warning.getUnconfirmedDeliveries().forEach(d -> {
                log.error("[NOTIFICATION-ACTOR] Unconfirmed Messages: {}", d.message());

                confirmMessage(new MsgConfirmed(d.deliveryId()));
            });
        } else {
            unhandled(msg);
        }
    }

    private void deliverAckMessage(NotificationMessage event) {
        deliver(notificationWorkers.path(), (Function<Long, Object>) deliveryId -> new AckMessage(deliveryId, event));
    }

    private void confirmMessage(final MsgConfirmed evt) {
        confirmDelivery(evt.deliveryId);
        deleteMessages(evt.deliveryId);
    }

    public interface NotificationMessage extends Event {}

    public static final @Data class PushMessage implements NotificationMessage {
        private final Long source;
        private final Long target;
        private final String trigger;
        private final ImmutableMap<String, Serializable> payload;
    }
}

public class NotificationWorkerActor extends UntypedActor {
    private final LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    private final @NonNull NotificationService notificationService;

    @Autowired
    public NotificationWorkerActor(final NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @Override public void onReceive(final Object event) throws Throwable {
        if (event instanceof AckMessage) {
            final AckMessage ackMessage = (AckMessage) event;
            NotificationMessage msg = (NotificationMessage) ackMessage.msg;
            log.info("[NOTIFICATION] receive message: {}", msg);

            if (msg instanceof PushMessage) {
                final PushMessage m = (PushMessage) msg;
                log.info("[NOTIFICATION] send push notification from: {} to: {}", m.getSource(), m.getTarget());
                notificationService.notify(m.getSource(), m.getTarget(), m.getTrigger(), m.getPayload());
            }
            sender().tell(new Confirm(ackMessage.deliveryId), self());
        } else {
            unhandled(event);
        }
    }
}

public class OrderService {
    public void createOrder() {
        actorSystemManager.notificationActor.tell(
          new PushMessage(), ActorRef.noSender()
        );
    }
}
```

最早实施这个方案的时候遇到一个问题，说一下这个问题如何产生的。我们一共有三台服务器，三台服务器都会部署同样的代码，以NotificationActor为例，它会分别部署在三个机器上。actor journal我们使用mysql存储。akka persistent actor内部有一个sequence number用来对接收到的消息进行计数，这个数字是递增的。同时这个数字也会在journal中记录。最初我的persistenceId方法是这样实现的：
```
@Override public String persistenceId() {
    return "journal:notification-actor";
}
```

那么，假如server1上的NotificationActor接收了一个消息，那么它的sequence number会变成1，mysql中将会存储的sequence number为1的消息。这时server2上也接收到了一个消息，因为它的最初sequence number是0，所以它也会把现在接收到的消息的sequence number设置为1。但是显然这条消息是不能持久化的，因为它和数据库记录的sequence number冲突了。根本原因是三台服务器上的NotificationActor的persistenceId是一样的。

上边代码中给出了一种方案，把persistenceId变成random的，每次actor启动的时候都会得到不同的persistenceId，这样就解决了上述问题。还有一种方案是引入akka cluster，使用akka singleton。这种方案会在下一篇文章中详细说明。

---
write on 2017-1-7