API里面参数声明为Body

```java
@POST("inquiryOrder/creatInquiry")
    Call<ResultBean<String>> creatInquiry(@Header("Authorization") String token, @Body RequestBody body);
```

工具类

```java
public class RequestBodyUtil {

    public static RequestBody getRequestBody(String[] key, Object[] value) {
        JSONObject root = new JSONObject();
        for (int a = 0; a < key.length; a++) {
            try {
                root.put(key[a], value[a]);
            } catch (JSONException e) {
                e.printStackTrace();
            }
        }
        String s = root.toString();
        return RequestBody.create(MediaType.parse("application/json"), s);
    }

    public static RequestBody getRequestBody(Object object) {
        Gson gson = new Gson();
        String s = gson.toJson(object);
        Log.d("RequestBodyUtil",s);
        return RequestBody.create(MediaType.parse("application/json"), s);
    }
}
```

