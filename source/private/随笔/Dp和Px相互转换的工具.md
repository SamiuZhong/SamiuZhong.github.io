```
public class DpUtils {

    private static TypedValue mTmpValue = new TypedValue();

    /**
     * 功能描述：dp转 px
     * 作者：YangShiJie
     * 创建时间：2019/7/15：12:23
     **/
    public static int Dp2Px(Context context, float dp) {
        final float scale = context.getResources().getDisplayMetrics().density;
        return (int) (dp * scale + 0.5f);
    }

    /**
     * 功能描述：px 转 dp
     * 作者：YangShiJie
     * 创建时间：2019/7/15：12:23
     **/
    public static int Px2Dp(Context context, float px) {
        final float scale = context.getResources().getDisplayMetrics().density;
        return (int) (px / scale + 0.5f);
    }

    /**
     * 功能描述：获取 资源id
     * 作者：YangShiJie
     * 创建时间：2019/7/15：12:24
     **/
    public static int getXmlDef(Context context, int id) {
        synchronized (mTmpValue) {
            TypedValue value = mTmpValue;
            context.getResources().getValue(id, value, true);
            return (int) TypedValue.complexToFloat(value.data);
        }
    }
}
```
