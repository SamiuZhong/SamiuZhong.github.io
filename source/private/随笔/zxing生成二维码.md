Github: [zxing](https://github.com/zxing/zxing)

参考文章：

 [Android扫一扫：zxing的集成与优化](https://www.jianshu.com/p/a4ba10da4231)

[Zxing简单集成](https://www.jianshu.com/p/60820c83fcb7)

## 正文

首先引用两个包

```groovy
api 'com.google.zxing:android-core:3.3.0'
api 'com.google.zxing:core:3.3.2'
```

Activity里面

```java
private static int size = 1000;
private ImageView mImageView;

@Override
public void initData() {
    binding.toolbarImg.setOnClickListener(v -> finish());
    mImageView = findViewById(R.id.display);
    createQrCode();
}

private void createQrCode() {
    new Thread(() -> {
        Map<String, Object> codeMap = new HashMap<>();
        codeMap.put("qrCodeType", "user");
        codeMap.put("id", "1234567890");
        codeMap.put("createTime", System.currentTimeMillis());
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.default_avatar);
        Bitmap qrCode = createQRCodeWithLogo(QRCodeActivity.this, GsonUtils.toGson(codeMap), bitmap);
        runOnUiThread(() -> mImageView.setImageBitmap(qrCode));
    }).start();
}

private Bitmap code(String content) {
    Hashtable<EncodeHintType, Object> hints = new Hashtable<>();
    hints.put(EncodeHintType.CHARACTER_SET, "UTF-8"); // 字符转码格式设置
    hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.L); // 容错级别设置 默认为L
    hints.put(EncodeHintType.MARGIN, "0"); // 空白边距设置
    BitMatrix bitMatrix;
    int[] pixels = new int[size * size];
    Bitmap bitmap = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888);
    try {
        bitMatrix = new QRCodeWriter().encode(content, BarcodeFormat.QR_CODE, size, size, hints);
        for (int y = 0; y < size; y++) {
            for (int x = 0; x < size; x++) {
                if (bitMatrix.get(x, y)) { // 黑色色块像素设置
                    pixels[y * size + x] = getResources().getColor(R.color._FF6A87);
                } else { // 白色色块像素设置
                    pixels[y * size + x] = getResources().getColor(R.color._FFFFFF);
                }
            }
        }
        bitmap.setPixels(pixels, 0, size, 0, 0, size, size);
    } catch (WriterException e) {
        e.printStackTrace();
    }
    return bitmap;
}

/**
 * 生成带logo的二维码，logo默认为二维码的1/5
 *
 * @param text 需要生成二维码的文字、网址等
 * @return bitmap
 */
public static Bitmap createQRCodeWithLogo(Context context, String text, Bitmap logoBitmap) {
    try {
        int MY_WIDTH = size / 10;
        Hashtable<EncodeHintType, Object> hints = new Hashtable<>();
        hints.put(EncodeHintType.CHARACTER_SET, "utf-8");
        /*
         * 设置容错级别，默认为ErrorCorrectionLevel.L
         * 因为中间加入logo所以建议你把容错级别调至H,否则可能会出现识别不了
         */
        hints.put(EncodeHintType.MARGIN, "0"); // 空白边距设置
        hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.H);
        BitMatrix bitMatrix = new QRCodeWriter().encode(text, BarcodeFormat.QR_CODE, size, size, hints);
        int width = bitMatrix.getWidth();//矩阵高度
        int height = bitMatrix.getHeight();//矩阵宽度
        int halfW = width / 2;
        int halfH = height / 2;
        Matrix m = new Matrix();
        float sx = (float) 2 * MY_WIDTH / logoBitmap.getWidth();
        float sy = (float) 2 * MY_WIDTH / logoBitmap.getHeight();
        m.setScale(sx, sy);
        //设置缩放信息
        //将logo图片按martix设置的信息缩放
        logoBitmap = Bitmap.createBitmap(logoBitmap, 0, 0,
                logoBitmap.getWidth(), logoBitmap.getHeight(), m, false);
        int[] pixels = new int[size * size];
        for (int y = 0; y < size; y++) {
            for (int x = 0; x < size; x++) {
                if (x > halfW - MY_WIDTH && x < halfW + MY_WIDTH
                        && y > halfH - MY_WIDTH
                        && y < halfH + MY_WIDTH) {
                    //该位置用于存放图片信息
                    //记录图片每个像素信息
                    pixels[y * width + x] = logoBitmap.getPixel(x - halfW + MY_WIDTH, y - halfH + MY_WIDTH);
                } else {
                    if (bitMatrix.get(x, y)) {
                        pixels[y * size + x] = context.getResources().getColor(R.color._FF6A87);
                    } else {
                        pixels[y * size + x] = context.getResources().getColor(R.color._FFFFFF);
                    }
                }
            }
        }
        Bitmap bitmap = Bitmap.createBitmap(size, size,
                Bitmap.Config.ARGB_8888);
        bitmap.setPixels(pixels, 0, size, 0, 0, size, size);
        return bitmap;
    } catch (WriterException e) {
        e.printStackTrace();
        return null;
    }
}
```



