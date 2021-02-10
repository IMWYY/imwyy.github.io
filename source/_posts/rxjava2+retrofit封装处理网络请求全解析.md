---
title: rxjava2+retrofit封装处理网络请求全解析
date: 2017-12-22 10:49:07
tags: Android
---

使用rxjava2+retrofit处理网络请求，线程的切换变得十分简单，代码也简洁了很多。但是简介的代码就是对可扩展性有着负面的影响，所以要对rxjava2+retrofit进行一定封装，使结构更清晰，可扩展性更强。这里给出一种可行的封装。

以下均以登陆请求为例子。

- API地址：`http://xxx/user/login`
- Post请求，参数`account`和`password`均为String

## 简单封装
1. **首先定一个retrofit接口：`UserService`**

	```java
	public interface UserService {

	    @FormUrlEncoded
	    @POST("/user/login")
	    Observable<ResponseBean<UidBean>> login(@Field("account") String account,
	                                            @Field("password") String password);
	}
	```

	这里`ResponseBean`是一个返回值json对应的实体类，服务器返回的json数据格式为`{code: xxx, message: xxx, data:xxx}`。而`UidBean`是一个data的具体内容，也是个json实体类，具体是什么无关紧要。

	```java
	/**
	 * 服务器返回值的实体类
	 * Created by stephen on 2017/9/8.
	 */

	public class ResponseBean<T> {
	    @SerializedName("code")
	    private int code;

	    @SerializedName("message")
	    private String message;

	    @SerializedName("data")
	    private T data;

	    public ResponseBean(int code, String message) {
	        this.code = code;
	        this.message = message;
	    }

	    public int getCode() {
	        return code;
	    }

	    public String getMessage() {
	        return message;
	    }

	    public T getData() {
	        return data;
	    }

	    /**
		 * 根据返回码确定API是否请求返回失败
		 *
		 * @return 失败返回true, 成功返回false
		 */
		public boolean isCodeValid() {
		    return code == 0;
		}

	    @Override
	    public String toString() {
	        return "code:" + code + ", message:" + message + ",data:" + data.toString();
	    }
	}
	```
2. **Oberser类的封装**

	这里`BaseObserver`里直接对`Disposable`进行处理，内存泄漏什么的省心多了。同时还保存了一个子类可见的errMsg，可以获取自定义的错误类型。

	（关于Disposable可以参考我的[这篇文章](https://blog.csdn.net/c_j33/article/details/78774546)，如果有哪里理解错了记得帮忙指出哦）。

	```java
	public abstract class BaseObserver<T> implements Observer<T> {

	    protected String errMsg = "";
	    private Disposable disposable;

	    @Override
	    public void onSubscribe(Disposable d) {
	        disposable = d;
	    }

	    @Override
	    public void onNext(T t) {}

	    @Override
	    public void onError(Throwable e) {
	        LogUtils.e("Observer.java", e.getMessage() + " " + toastErrHint);

	        if (!NetworkUtils.isConnected()) {
	            errMsg = "网络连接出错,";
	        } else if (e instanceof HttpException) {
	            errMsg = "网络请求出错,";
	        } else if (e instanceof IOException) {
	            errMsg = "网络出错,";
	        }

	        disposeIt();
	    }

	    @Override
	    public void onComplete() {
	        disposeIt();
	    }

	    /**
	     * 销毁disposable
	     */
	    private void disposeIt() {
	        if (disposable != null && !disposable.isDisposed()) {
	            disposable.dispose();
	            disposable = null;
	        }
	    }
	}
	```

3. **接口类整合retrofit**

	- `HttpFactory`类里是一些网络交互的基本设置，单独放一个类是为了方便扩展。之后就会发现这样做的好处。

	```java
	public class HttpFactory {

	    private static HttpFactory instance;
	    public synchronized static HttpFactory getInstance() {
	        return instance == null ? instance = new HttpFactory() : instance;
	    }

	    private Retrofit mRetrofit;

	    private HttpFactory() {
	        //创建一个OkHttpClient并设置超时时间
	        OkHttpClient httpClient = new OkHttpClient.Builder()
	                .connectTimeout(AppConfig.DEFAULT_TIMEOUT, TimeUnit.SECONDS)
	                .cookieJar(cookieJar)
	                .build();

	        mRetrofit = new Retrofit.Builder()
	                .client(httpClient)
	                .baseUrl(AppConfig.URL_BASE)
	                .addConverterFactory(GsonConverterFactory.create())
	                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
	                .build();
	    }

	    Retrofit getmRetrofit() {
	        return mRetrofit;
	    }
	}
	```

	- `UserApi`类将retrofit接口封装成rxjava调用的方法。

	```java
	public class APIUser extends BaseAPI {
	    private static final APIUser INSTANCE = new APIUser();

	    public static APIUser getInstance() {
	        return INSTANCE;
	    }

	    private UserService userService;
	    private Retrofit mRetrofit;

	    private APIUser() {
	        mRetrofit = HttpFactory.getInstance().getmRetrofit();
	        userService = mRetrofit.create(UserService.class);
	    }

	    /**
	     * 登录 这里为了方便逻辑展示，password直接明文传输
	     *
	     * @param account  账号
	     * @param password 密码
	     * @param observer 观察者
	     */
	    public void login(String account, String password, BaseObserver<UidBean> observer) {
	        userService.login(account, password)
	                .subscribeOn(Schedulers.io())
	                .observeOn(Schedulers.computation())
	                .map(new Function<ResponseBean<UidBean>, UidBean>() {
	                    @Override
	                    public UidBean apply(ResponseBean<UidBean> responseBean) throws Exception {
	                        return responseBean.getData();
	                    }
	                })
	                .observeOn(AndroidSchedulers.mainThread())
	                .subscribe(observer);
	    }

	}
	```

4. **愉快的调用**

	```java
	APIUser.getInstance().login(account, password, new BaseObserver<UidBean>() {
	    @Override
	    public void onNext(UidBean uidBean) {
	    	// your logic code here
	        ToastUtils.showShort("登录成功");
	    }
	});
	```

## 添加cookie支持

以上实现了一个简单的封装，现在我们开始加入cookie支持：第一次拿到set-cookie的时候保存cookie，然后再每次request的时候加入cookie。

这里借用了一下GitHub上别人造的轮子，使用了[PersistentCookieJar](https://github.com/franmontiel/PersistentCookieJar)。`PersistentCookieJar`是`ClearableCookieJar`的一个具体实现，利用SharedPreferences保存cookie。具体使用方法如下：

- 添加gradle依赖
- 在`HttpFactory`添加cookieJar

```java
public class HttpFactory {

    private static HttpFactory instance;
    public synchronized static HttpFactory getInstance() {
        return instance == null ? instance = new HttpFactory() : instance;
    }

    private Retrofit mRetrofit;
    private ClearableCookieJar cookieJar;

    private HttpFactory() {
        //设置cookie
        cookieJar = new PersistentCookieJar(new SetCookieCache(), new SharedPrefsCookiePersistor());
        //创建一个OkHttpClient并设置超时时间
        OkHttpClient httpClient = new OkHttpClient.Builder()
                .connectTimeout(AppConfig.DEFAULT_TIMEOUT, TimeUnit.SECONDS)
                .cookieJar(cookieJar)
                .build();

        mRetrofit = new Retrofit.Builder()
                .client(httpClient)
                .baseUrl(AppConfig.URL_BASE)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
    }

    Retrofit getmRetrofit() {
        return mRetrofit;
    }

    /**
     * 退出登录后清除cookie
     */
    public void clearCookie() {
        cookieJar.clear();
    }
}
```

这里`HttpFactory`还提供了一个对外接口，可以在退出登录后调用清除`clearCookie()`cookie。

**现在每次处理请求的时候，`ClearableCookieJar`就会帮你打点好所有的cookie验证相关信息。**

## 添加错误码统一处理

很多时候服务器返回的json数据格式会有一个统一的格式，比如以上说的形式：`{code: xxx, message: xxx, data:xxx}`。带有返回状态码和错误信息，如果对于每个code我们都在`Observer`中的`onNext`处理，显然很麻烦很冗余一点也不优雅，可扩展性也很差。比如服务器新添加一个状态码，所有的代码都要改，不切实际嘛。所以我们下面要做的就是**对错误码添加统一处理。**

实现思路：对于json数据response的时候，我们都会用`GsonConverterFactory`来将json转换为java对象，那么我们就可以利用`GsonConverterFactory`来统一处理。我们来重新定义一个类实现`GsonConverterFactory`的关键方法。因为`GsonConverterFactory`是final的，不可继承，所以这里会重新定义三个类：`CustomGsonConverterFactory`，`CustomGsonResponseBodyConverter`和`CustomGsonRequestBodyConverter`分别对应默认的`GsonConverterFactory`，`GsonResponseBodyConverter`和`GsonRequestBodyConverter`。

- CustomGsonConverterFactory

	```java
	/**
	 * 自定义的GsonConverterFactory
	 * 主要是重写了responseBodyConverter类
	 * Created by stephen on 2017/10/2.
	 */
	public final class CustomGsonConverterFactory extends Converter.Factory {
	    /**
	     * Create an instance using a default {@link Gson} instance for conversion. Encoding to JSON and
	     * decoding from JSON (when no charset is specified by a header) will use UTF-8.
	     */
	    public static CustomGsonConverterFactory create() {
	        return create(new Gson());
	    }

	    /**
	     * Create an instance using {@code gson} for conversion. Encoding to JSON and
	     * decoding from JSON (when no charset is specified by a header) will use UTF-8.
	     */
	    public static CustomGsonConverterFactory create(Gson gson) {
	        return new CustomGsonConverterFactory(gson);
	    }

	    private final Gson gson;

	    private CustomGsonConverterFactory(Gson gson) {
	        if (gson == null) throw new NullPointerException("gson == null");
	        this.gson = gson;
	    }

	    @Override
	    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
	        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
	        return new CustomGsonResponseBodyConverter<>(gson, adapter);
	    }

	    @Override
	    public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations,
	                                                          Annotation[] methodAnnotations, Retrofit retrofit) {
	        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
	        return new CustomGsonRequestBodyConverter<>(gson, adapter);
	    }
	}
	```
- CustomGsonRequestBodyConverter。这个类没有改动，直接copy了GsonRequestBodyConverter的源码。

	```java
	final class CustomGsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
	    private static final MediaType MEDIA_TYPE = MediaType.parse("application/json; charset=UTF-8");
	    private static final Charset UTF_8 = Charset.forName("UTF-8");

	    private final Gson gson;
	    private final TypeAdapter<T> adapter;

	    CustomGsonRequestBodyConverter(Gson gson, TypeAdapter<T> adapter) {
	        this.gson = gson;
	        this.adapter = adapter;
	    }

	    @Override public RequestBody convert(T value) throws IOException {
	        Buffer buffer = new Buffer();
	        Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
	        JsonWriter jsonWriter = gson.newJsonWriter(writer);
	        adapter.write(jsonWriter, value);
	        jsonWriter.close();
	        return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
	    }
	}
	```
- CustomGsonResponseBodyConverter。重信写`convert`方法。这里直接将返回值转换为responseBean，并根据返回值判断是否返回成功（这里根据`responseBean.isCodeValid()`判断）。如果返回不成功，可以自定义一些exception，并抛出异常。比如我这里的`throw new APIException`。

	```java
	final class CustomGsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
	    private final Gson gson;
	    private final TypeAdapter<T> adapter;
	    private static final Charset UTF_8 = Charset.forName("UTF-8");

	    CustomGsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
	        this.gson = gson;
	        this.adapter = adapter;
	    }

	    @Override
	    public T convert(ResponseBody value) throws IOException {
	        String response = value.string();
			//关键代码
	        ResponseBean responseBean = gson.fromJson(response, ResponseBean.class);
	        if (!responseBean.isCodeValid()) {
	            value.close();
	            throw new APIException(responseBean.getCode(), responseBean.getMessage());
	        }

	        MediaType contentType = value.contentType();
	        Charset charset = contentType != null ? contentType.charset(UTF_8) : UTF_8;
	        InputStream inputStream = new ByteArrayInputStream(response.getBytes());
	        Reader reader = new InputStreamReader(inputStream, charset);
	        JsonReader jsonReader = gson.newJsonReader(reader);

	        try {
	            return adapter.read(jsonReader);
	        } finally {
	            value.close();
	        }
	    }
	}
	```

- 同样的自定义的exception可以在`BaseObserver`的`onError`中捕获然后获取不同的`errMsg`。

	```java
	@Override
	public void onError(Throwable e) {
	    LogUtils.e("Observer.java", e.getMessage() + " " + toastErrHint);

	    if (!NetworkUtils.isConnected()) {
	        errMsg = "网络连接出错,";
	    } else if (e instanceof APIException) {
	        APIException exception = (APIException) e;
	        errMsg = exception.getMessage();
	    } else if (e instanceof HttpException) {
	        errMsg = "网络请求出错,";
	    } else if (e instanceof IOException) {
	        errMsg = "网络出错,";
	    }
	}
	```

## 总结

好了至此，封装已经可以达到了一定的可扩展性，使用也很方便。事实上你还可以在`BaseObserver`中添加一些其他的东西，比如加载动画等等。

大家有意见提出来一起探讨，共同进步。:）