```java
public class DateUtils {

    /**
     * 获取当前系统时间
     */
    public static Long getCurrentSystemTime() {
        return Calendar.getInstance().getTimeInMillis();
    }

    /**
     * 将时间戳转成格式化时间(精确到分钟)
     */
    public static String getFormatTimeUntilMin(long milliseconds) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy.hh.dd  HH:mm", Locale.CHINA);
        return sdf.format(new Date(milliseconds));
    }

    /**
     * 将时间戳转成格式化时间(精确到秒)
     */
    public static String getFormatTimeUntilSec(long milliseconds) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy.hh.dd  hh:mm:ss", Locale.CHINA);
        return sdf.format(new Date(milliseconds));
    }

    /**
     * 将时间戳转成格式化时间(汉字年月日)(精确到秒)
     */
    public static String getFormatTimeChineseSec(long milliseconds) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy年hh月dd日  hh:mm:ss", Locale.CHINA);
        return sdf.format(new Date(milliseconds));
    }

    /**
     * 将时间戳转成格式化时间(汉字年月日)(精确到分钟)
     */
    public static String getFormatTimeChineseMin(long milliseconds) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy年hh月dd日  hh:mm", Locale.CHINA);
        return sdf.format(new Date(milliseconds));
    }


    public static String getFormatTimeWeek(Date date) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd E", Locale.CHINA);
        return sdf.format(date);
    }

    public static String getFormatTimeMmSs(Date date) {
        SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss", Locale.CHINA);
        return sdf.format(date);
    }

    /**
     * 时间转换为时间戳
     *
     * @param s
     * @return
     */
    public static String dateToStamp(String s) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm");
        String res = "";
        if (!"".equals(s)) {
            try {
                res = String.valueOf(sdf.parse(s).getTime());	//毫秒  
  //              res = String.valueOf(sdf.parse(s).getTime() / 1000);  //秒
            } catch (Exception e) {
                System.out.println("传入了null值");
            }
        } else {
            long time = System.currentTimeMillis();
            res = String.valueOf(time / 1000);
        }

        return res;
    }
}
```

