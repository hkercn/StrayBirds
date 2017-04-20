---
layout: post
title: 视频直播项目总结之MINA2使用总结
category: 技术
comments: false
---

## 基础知识 ##

关于NIO的基础理论知识，推荐 [Java NIO: Non-blocking Server非阻塞服务器](https://java-nio.avenwu.net/java-nio-non-blocking-server.html)

关于mina2的基础理论知识，推荐waylau童鞋翻译的 [Apache MINA 2 用户指南](https://www.gitbook.com/book/waylau/apache-mina-2-user-guide/details).

## 应用总结 ##

### 客户端基本配置 ###

#### 会话连接的配置与创建 ####
```java

//创建TCP客户端连接器
connector = new NioSocketConnector();
//设置超时
connector.setConnectTimeoutMillis(YayaConstant.reqTimeoutMillSeconds);
//设置接收缓冲区的大小
connector.getSessionConfig().setReceiveBufferSize(1024*32);   
//设置输出缓冲区的大小
connector.getSessionConfig().setSendBufferSize(1024 * 32);
//读写通道的空闲状态的超时时间
connector.getSessionConfig().setIdleTime(IdleStatus.BOTH_IDLE, IDELTIMEOUT);
//设置保持活动状态
connector.getSessionConfig().setKeepAlive(true);
//设置读通道的闲置时间
connector.getSessionConfig().setReaderIdleTime(5);
//设置为非延迟发送，为true则不组装成大包发送，收到东西马上发出 
connector.getSessionConfig().setTcpNoDelay(true);
//ProtocolCodecFilter这个过滤器将会把二进制或者协议特定的数据翻译为消息对象，反之亦然
connector.getFilterChain().addLast("codec", new ProtocolCodecFilter(new ProxyCoderFactory()));

//设置tcp心跳
KeepAliveMessageFactoryImpl heartBeatFactory = new KeepAliveMessageFactoryImpl();
KeepAliveFilter heartBeat = new KeepAliveFilter(heartBeatFactory,IdleStatus.BOTH_IDLE, KeepAliveRequestTimeoutHandler.CLOSE);
heartBeat.setForwardEvent(false);//继续调用 IoHandlerAdapter 中的 sessionIdle时间
heartBeat.setRequestInterval(IDELTIMEOUT);//设置当连接的读取通道空闲的时候，心跳包请求时间间隔
heartBeat.setRequestTimeout(HEARTBEATRATE);//设置心跳包请求后 等待反馈超时时间。 超过该时间后则调用KeepAliveRequestTimeoutHandler.CLOSE
connector.getFilterChain().addLast("heartbeat", heartBeat);
//自定义日志过滤器
LoggingFilter loggingFilter = new LoggingFilter();
loggingFilter.setSessionClosedLogLevel(LogLevel.NONE);
loggingFilter.setSessionCreatedLogLevel(LogLevel.NONE);
loggingFilter.setSessionOpenedLogLevel(LogLevel.NONE);
loggingFilter.setMessageReceivedLogLevel(LogLevel.NONE);
loggingFilter.setMessageSentLogLevel(LogLevel.NONE);
connector.getFilterChain().addLast("logger", loggingFilter);
//设置处理器ProxyIoHanderAdapter
connector.setHandler(new ProxyIoHanderAdapter());

long firstConnectedTimeMillis = System.currentTimeMillis();
for (;;) {//死循环直至会话创建并连接成功
  try {
    //设置服务器的主机名和端口号
    inetSocketAddress = new InetSocketAddress(host,port);
    //连接服务器
    ConnectFuture future = connector.connect(inetSocketAddress);
    //Wait until the connection attempt is finished.阻塞方式等待IOSESSION的创建完成
    future.awaitUninterruptibly();
    //获取IoSession
    ioSession = future.getSession();
    if(ioSession.isConnected()){
      Log.d(TAG, "连接成功");
      break;
    }
  } catch (Exception e) {
    Log.d(TAG, "连接异常");
    long currentTimeMillis = System.currentTimeMillis();
    // 重连前30秒无须等待5秒
    if(currentTimeMillis - firstConnectedTimeMillis > 1000 * 30) {
      try {
        Thread.sleep(5000);
      } catch (InterruptedException e1) {
        e1.printStackTrace();
      }
    }
  }
}

```

其中ProxyIoHanderAdapter为继承自IOHanderAdapter的自定义处理器类，其关键的函数回调messageReceived代码如下

```java

@Override
public void messageReceived(IoSession session, Object message) throws Exception {
  if (firstRunFlag) {
    firstRunFlag = false;
    secretKey = message.toString();
    EventBus.getDefault().post(new TcpSecretKey(message.toString()));
  } else {
    EventBus.getDefault().post((TlvSignal) message);
  }
}

```
firstRunFlag默认值为true，标识会话接收到的message是否是session创建成功后接收到的第一条message。

如果是，那么通过EventBus全局发送TCP会话成功创建并连接的通知事件TcpSecretKey。此时接收到的message为后续客户端读写服务器数据流时需要的secret-key信息。
TcpSecretKey类结构也比较单一，就是一个包含了secret-key信息的数据实体类

如果不是，那么说明此次接收到的message是客户端写数据流到客户端后的服务器响应数据，进而协议解码全局转发。

数据流的读写，操作的是自实现数据协议Tlv的基础对象TlvSignal，下面会着重介绍下该协议内容。

#### 发送数据 ####

```java

/**
 * 写数据
 *
 * @param tlvSignal
 */
public void write(TlvSignal tlvSignal) {
  Log.d(TAG,"TCPConection-write:tlvSignal:"+tlvSignal.toString());
  try {
    if (secretKey == null) {//连接还未创建或成功连接或者断开连接了
      if(tlvSignal.getRetureMsgCode() != 0 && tlvSignal.getReturnModuleTag() != 0){
        defaultResp(tlvSignal.getReturnModuleTag(),tlvSignal.getRetureMsgCode(),
            YayaConstant.RESULT_NETWORK_ERROR,YayaConstant.RESULT_NETWORK_ERROR_MSG,tlvSignal.getUuid());
      }
    }
    if(ioSession != null && ioSession.isConnected()){
      //编码数据流并写入服务器
      WriteFuture wf = ioSession.write(TlvCodecUtil.encodeSignal(secretKey.getBytes(), tlvSignal,YayaService.tlvStore2));
      //等待发送数据操作完成，超时时间10s
      wf.awaitUninterruptibly(10000);
      if(wf.isWritten()){
        //发送成功
        return;
      }else{
        //发送失败处理
        if(tlvSignal.getRetureMsgCode() != 0 && tlvSignal.getReturnModuleTag() != 0){
          defaultResp(tlvSignal.getReturnModuleTag(),tlvSignal.getRetureMsgCode(),
              YayaConstant.RESULT_TIMEOUT,YayaConstant.RESULT_TIMEOUT_MSG,tlvSignal.getUuid());
        }
      }
    }else{
      if(tlvSignal.getRetureMsgCode() != 0 && tlvSignal.getReturnModuleTag() != 0){
        defaultResp(tlvSignal.getReturnModuleTag(),tlvSignal.getRetureMsgCode(),
            YayaConstant.RESULT_NETWORK_ERROR,YayaConstant.RESULT_NETWORK_ERROR_MSG,tlvSignal.getUuid());
      }
    }
  } catch (Exception e) {
    e.printStackTrace();
    if(tlvSignal.getRetureMsgCode() != 0 && tlvSignal.getReturnModuleTag() != 0){
      defaultResp(tlvSignal.getReturnModuleTag(),tlvSignal.getRetureMsgCode(),
          YayaConstant.RESULT_NETWORK_ERROR,YayaConstant.RESULT_NETWORK_ERROR_MSG,tlvSignal.getUuid());
    }
  }
}

/**
 * 发送失败时，失败信息经由默认协议发送错误码
 */
@SuppressWarnings("unchecked")
private void defaultResp(byte moduleId,int msgCode,Long resultCode,String resultMsg,String uuid){
  Log.w(TAG,"TCPConection-defaultResp timeout:moudleid:"+moduleId+" msgCode:"+msgCode+" resultMsg:"+resultMsg + " uudi:"+uuid);
  TlvSignal tlvSignal;
  Class type = YayaService.tlvStore2.getTypeMeta(moduleId,msgCode);
  if (null != type) {
    try {
      tlvSignal = TlvCodecUtil.decodeTlvSignal("timeout".getBytes(),
          type,YayaService.tlvStore2.getTlvFieldMeta(type),YayaService.tlvStore2);
      if(StringUtils.isNotEmpty(tlvSignal.getUuid())){
        tlvSignal.setUuid(tlvSignal.getUuid());
      }
      tlvSignal.setResultCode(resultCode);
      tlvSignal.setResultMsg(resultMsg);
      tlvSignal.setUuid(uuid);

      EventBus.getDefault().post(tlvSignal);
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}

```

TlvCodecUtil为写数据到服务器/从服务器接受数据的过程中，需要的数据编解码工具类。YayaService.tlvStore2是一个管理类，
其内部map类型的字段typeMetaCache，用来存储所有业务逻辑需要的协议消息对象。

在App启动之初，Application中会启动并初始化服务YayaService，而YayaService的职责，就是初始化tlvStore并紧接着注册所有需要的event。

```java

public class YayaService extends Service {
	private static final String TAG = YayaService.class.getSimpleName();

	@Override
	public IBinder onBind(Intent intent) {
		Log.d(TAG,"onBind");
		return null;
	}

	@Override
	public void onCreate() {
		super.onCreate();
		new InitAsyncTask(this).execute();
	}
  
	private static class InitAsyncTask extends AsyncTask<Void, Void, Void> {
		private WeakReference<YayaService> mService;
    
		public InitAsyncTask (YayaService service) {
			mService = new WeakReference<YayaService>(service);
		}

		@Override
		protected Void doInBackground(Void... params) {
			final YayaService service = mService.get();
			if (service != null) {
				Log.d(TAG, "InitAsyncTask-YayaService.initTlvStore()");
				service.initTlvStore();
			}
			return null;
		}

		@Override
		protected void onPostExecute(Void aVoid) {
			super.onPostExecute(aVoid);
			final YayaService service = mService.get();
			if (service != null) {
				service.postInitializeEvent();
			}
			mService.clear();
		}
	}
  
	private final Handler mHandler = new Handler(Looper.getMainLooper());

	private void postInitializeEvent () {
		mHandler.post(new Runnable() {
			@Override
			public void run() {
				initEvents();
			}
		});
	}

	private void initEvents(){
		//注册并初始化所有业务逻辑的请求与相应事件-EventBus
    //code...
	}

	@Override
	public void onDestroy() {
		//退出activity
		YayaApplication.exit();
		super.onDestroy();
	}

	@Override
	public int onStartCommand(Intent intent, int flags, int startId) {
		return START_NOT_STICKY;
	}


	/**
	 * 初始化tlvstore
	 */
	private void initTlvStore(){
		try {
			if (tlvStore2 == null) {
				tlvStore2 = TlvUtil.initialTlvStore();
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}

```

TlvUtil类则是Tlv协议的基础操作工具类，包含了对Tlv协议的初始化、读写字段等操作。

### Tlv通信协议 ###

#### 1. 什么是Tlv协议? ####

   [自定义通信协议设计之TLV编码应用](https://my.oschina.net/maxid/blog/206546)

   [类型-长度-值（TLV）协议](http://wizmann.tk/tlv-protocol.html)

   [TLV编码通信协议设计](http://www.wtango.com/tlv%E7%BC%96%E7%A0%81%E9%80%9A%E4%BF%A1%E5%8D%8F%E8%AE%AE%E8%AE%BE%E8%AE%A1/)
   
#### 2.相关知识点 ####

   [Java中的Type详解](http://loveshisong.cn/%E7%BC%96%E7%A8%8B%E6%8A%80%E6%9C%AF/2016-02-16-Type%E8%AF%A6%E8%A7%A3.html)
   
#### 3. 自实现Tlv协议综述 ####

"协议一般由一个或多个消息组成，简单的来说，消息就像是一个Table，由表头(消息的字段定义，包括名称与数据类型)与行(字段值)组成"

协议的超类Tlvable是一个空实现的接口，消息对象TlvSignal和消息头TlvAccessHeader实现自Tlvable。

为了更清晰的解读Tlv编码协议的封装使用，我们这里以逆向的思维，先从TlvSignal被编码为字节数组以便于通过MINA2写入会话连接的部分说起。

```java

WriteFuture wf = ioSession.write(TlvCodecUtil.encodeSignal(secretKey.getBytes(), tlvSignal,YayaService.tlvStore2));

```

TlvCodecUtil.encodeSignal方法的实现如下

```java

public static byte[] encodeSignal(byte[] key, TlvSignal tlvSignal,TlvStore tlvStore) {
	try {
		byte[] signalData = TlvCodecUtil.encodeTlvSignal(tlvSignal, tlvStore.getTlvFieldMeta(tlvSignal.getClass()), tlvStore);
		int signalDataLength = (null == signalData ? 0 : signalData.length);
		tlvSignal.getHeader().setLength(tlvSignal.getHeader().HEADER_LENGTH + signalDataLength);
		byte[] tlvHeaderData = encodeTlvHeader(tlvSignal.getHeader(), tlvStore);
		DataEncrypt.encrypt(YayaConstant.ENCRYPT_FLAG, key, signalData, signalData.length);
		return ArrayUtils.addAll(tlvHeaderData, signalData);
	} catch (IllegalAccessException e) {
		e.printStackTrace();
	} catch (Exception e) {
		e.printStackTrace();
	}
	return null;
}

```

TlvCodecUtil.encodeTlvSignal方法的实现如下

```java

public static byte[] encodeTlvSignal(Object object, Map<Integer, TlvFieldMeta> fieldMeta, TlvStore tlvStore) throws Exception {
	if (null == fieldMeta || fieldMeta.size() == 0) {
		return null;
	} else {
		byte[] data = null;
		for (Map.Entry<Integer, TlvFieldMeta> tlvFieldMetas : fieldMeta.entrySet()) {
			TlvFieldMeta tlvFieldMeta = tlvFieldMetas.getValue();

			tlvFieldMeta.getField().setAccessible(true);
			Object fieldValue = tlvFieldMeta.getField().get(object);

			if (null != fieldValue) {
				if (List.class.isAssignableFrom(tlvFieldMeta.getField().getType())) {
					List list = (List) fieldValue;
					for (Object o : list) {
						byte[] fieldData = (byte[]) tlvFieldMeta.getTransformer().encode(o, tlvFieldMeta, tlvStore);
						if (null != fieldData) {
							data = ArrayUtils.addAll(data, fieldData);
						}
					}
				} else {
					byte[] fieldData = (byte[]) tlvFieldMeta.getTransformer().encode(fieldValue, tlvFieldMeta, tlvStore);
					if (null != fieldData) {
						data = ArrayUtils.addAll(data, fieldData);
					}
				}
			}
		}
		return data;
	}
}
	
```

在YayaService#initTlvStore方法中，我们通过调用

```java

TlvUtil.initialTlvStore();

```

TlvUtil.initialTlvStore方法中主要的任务是实例化DefaultTlvStore的实例YayaService#tlvStore2，

```java


DefaultTlvStore tlvStore = new DefaultTlvStore();
tlvStore.addTypeMetaCache(QHKERCNInfo.class);

```

DefaultTlvStore类实现了抽象接口TlvStore，其代码实现如下

```java

public class DefaultTlvStore implements TlvStore {
    private final String TAG = DefaultTlvStore.class.getSimpleName();
    private Map<Byte, Map<Integer, Class<? extends Tlvable>>> typeMetaCache = new HashMap<Byte, Map<Integer, Class<? extends Tlvable>>>();
    private Map<Class, Map<Integer, TlvFieldMeta>> tlvFieldMetaCache = new HashMap<Class, Map<Integer, TlvFieldMeta>>();
    private Map<Class, SortedSet<TlvHeaderFieldMeta>> tlvHeaderFieldMetaCache = new HashMap<Class, SortedSet<TlvHeaderFieldMeta>>();

    public int getStoreSize(){
        return  typeMetaCache.size();
    }

    public DefaultTlvStore() throws Exception {
        initialRegisterTlvHeaderFieldMeta();
    }

    public void initialRegisterTlvHeaderFieldMeta() throws Exception {
        registerTlvHeaderFieldMeta(TlvAccessHeader.class);
    }

    /**
     * 注册Tlv编码协议的协议头字段元信息
     * @param type 协议头字段的定义类
     * @throws Exception
     */
    public void registerTlvHeaderFieldMeta(Class type) throws Exception {
        SortedSet<TlvHeaderFieldMeta> tlvFieldMetas = new TreeSet<TlvHeaderFieldMeta>();
        //获取所有Class中定义的字段
        Field[] fields = type.getDeclaredFields();
        for (Field field : fields) {
            //如果该字段上声明了TlvHeaderField注解
            if (field.isAnnotationPresent(TlvHeaderField.class)) {
                TlvHeaderField tlvHeaderField = field.getAnnotation(TlvHeaderField.class);
                TlvHeaderFieldMeta tlvFieldMeta = new TlvHeaderFieldMeta();
                //获取注解中指定的unsinged、index、type等字段值
                tlvFieldMeta.setUnsigned(tlvHeaderField.unsigned());
                tlvFieldMeta.setIndex(tlvHeaderField.index());
                tlvFieldMeta.setConvertor(ConvertorFactory.build(field.getType()));
                tlvFieldMeta.setField(field);

                tlvFieldMetas.add(tlvFieldMeta);
            }
        }

        if (tlvFieldMetas.size() > 0) {
            tlvHeaderFieldMetaCache.put(type, tlvFieldMetas);
        }
    }

    @Override
    public Class getTypeMeta(/*int modify by hu 2014.05.12*/byte moduleId, int msgCode) {
        Map<Integer, Class<? extends Tlvable>> mappers = typeMetaCache.get(moduleId);
        if (null != mappers) {
            return mappers.get(msgCode);
        } else {
            return null;
        }
    }

    @Override
    public Map<Integer, TlvFieldMeta> getTlvFieldMeta(Class type) {
        return tlvFieldMetaCache.get(type);
    }

    @Override
    public SortedSet<TlvHeaderFieldMeta> getTlvHeaderFieldMeta(Class type) {
        return tlvHeaderFieldMetaCache.get(type);
    }


    private boolean checkSupportFieldType(Field field) throws Exception {
        Class type = field.getType();
        if (String.class.isAssignableFrom(type)) {
            return true;
        } else if (Byte.class.isAssignableFrom(type) || type.equals(byte.class)) {
            return true;
        } else if (Short.class.isAssignableFrom(type) || type.equals(short.class)) {
            return true;
        } else if (Character.class.isAssignableFrom(type) || type.equals(char.class)) {
            return true;
        } else if (Integer.class.isAssignableFrom(type) || type.equals(int.class)) {
            return true;
        } else if (Long.class.isAssignableFrom(type) || type.equals(long.class)) {
            return true;
        } else if (Float.class.isAssignableFrom(type) || type.equals(float.class)) {
            return true;
        } else if (Double.class.isAssignableFrom(type) || type.equals(double.class)) {
            return true;
        } else if (TlvSignal.class.isAssignableFrom(type)) {
            return true;
        } else if (List.class.isAssignableFrom(type)) {
            return true;
        } else if (type.isArray() && type.getComponentType().equals(byte.class)) {
            return true;
        }

        throw new Exception("field isn't support." + field);
    }

    public void registerTlvFieldMeta(Class<? extends TlvSignal> type) throws Exception {
        //Log.d(TAG,"registerTlvFieldMeta start");
        Map<Integer, TlvFieldMeta> tlvFieldMetas = new HashMap<Integer, TlvFieldMeta>();
        Field[] fields = type.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(TlvSignalField.class)) {
                checkSupportFieldType(field);
                TlvSignalField tlvSignalField = field.getAnnotation(TlvSignalField.class);
                TlvFieldMeta tlvFieldMeta = new TlvFieldMeta();
                if (List.class.isAssignableFrom(field.getType())) {
                    Type fc = field.getGenericType();
                    if (fc instanceof ParameterizedType) {
                        ParameterizedType pt = (ParameterizedType) fc;
                        Class genericClass = (Class) pt.getActualTypeArguments()[0]; //generic type
                        tlvFieldMeta.setTransformer(TransformerFactory.build(genericClass));
                    } else {
                        Log.e(TAG, "array list must generic,type:" + type);
                        throw new Exception("array list must generic.");
                    }
                } else {
                    tlvFieldMeta.setTransformer(TransformerFactory.build(field.getType()));
                }
                tlvFieldMeta.setField(field);
                tlvFieldMeta.setTag(tlvSignalField.tag());
                tlvFieldMeta.setUnsigned(tlvSignalField.unsigned());

                tlvFieldMetas.put(tlvSignalField.tag(), tlvFieldMeta);

                if (field.getType().isAnnotationPresent(TlvVoMsg.class)) {
                    registerTlvFieldMeta((Class<? extends TlvSignal>) field.getType());
                } else if (List.class.isAssignableFrom(field.getType())) {
                    Type fc = field.getGenericType();
                    if (fc instanceof ParameterizedType) {
                        ParameterizedType pt = (ParameterizedType) fc;
                        Class genericClass = (Class) pt.getActualTypeArguments()[0]; //generic type

                        if (genericClass.isAnnotationPresent(TlvVoMsg.class)) {
                            if (!tlvFieldMetaCache.containsKey(genericClass)) {
                                registerTlvFieldMeta((Class<? extends TlvSignal>) genericClass);
                            }
                        }
                    }
                }
            }
        }

        if (tlvFieldMetas.size() > 0) {
            tlvFieldMetaCache.put(type, tlvFieldMetas);
        }
    }

    public void addTypeMetaCache(Class<? extends Tlvable> type) {
        try{
            if (type.isAnnotationPresent(TlvMsg.class)) {
                TlvMsg tlvMsg = type.getAnnotation(TlvMsg.class);
                Map<Integer, Class<? extends Tlvable>> mappers = typeMetaCache.get(tlvMsg.moduleId());
                if (null == mappers) {
                    mappers = new HashMap<Integer, Class<? extends Tlvable>>();
                }
                mappers.put(tlvMsg.msgCode(), type);
                typeMetaCache.put(tlvMsg.moduleId(), mappers);
            }
            if (TlvSignal.class.isAssignableFrom(type)) {
                registerTlvFieldMeta((Class<? extends TlvSignal>) type);
            }
        } catch (Exception e){
            e.printStackTrace();
        }
    }
}

```

在DefaultTlvStore构造函数中，注册了Tlv编码协议的协议头字段元信息，并保存在tlvHeaderFieldMetaCache中。其中注解类TlvHeaderField定义如下

```java

/**
 * @interface 声明TlvHeaderField为注解类
 * @Retention 声明注解保留到哪个时间段
 * 	RetentionPolicy.RUNTIME 这种类型的Annotations将被JVM保留,所以他们能在运行时被JVM或其他使用反射机制的代码所读取和使用.
 *
 * @Target 指示注释类型所适用的程序元素的种类
 * 	ElementType.FIELD 指定TlvHeaderField注解类仅限用于定义字段
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(value = ElementType.FIELD)
public @interface TlvHeaderField {
	//定义int类型的index字段
	int index();
	//定义Unsigned类型的unsigned字段，默认值为枚举值Unsigned.NONE
	Unsigned unsigned() default Unsigned.NONE;
}

```


TlvHeaderFieldMeta类 定义相对简单

```java

public class TlvHeaderFieldMeta implements Comparable<TlvHeaderFieldMeta>{
	private Field field;
    private int index;
    private Unsigned unsigned;
    private Convertor convertor;
	
	//getter/setter
	
｝

```

DefaultTlvStore#addTypeMetaCache方法用来增加Tlvable子类的元信息定义，该类定义是关键，前后端业务逻辑层面通信，就是通过这里定义的moduleId和msgCode字段，来区分业务逻辑类型的。TlvMsg注解类定义如下

```java

@Retention(RetentionPolicy.RUNTIME)
@Target(value = ElementType.TYPE)
public @interface TlvMsg {
	byte moduleId();
	int msgCode();
}

```



