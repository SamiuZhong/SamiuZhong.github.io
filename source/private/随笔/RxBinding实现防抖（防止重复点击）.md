```java
       RxView.clicks(btnLogin)
            .throttleFirst(2, TimeUnit.SECONDS)
            .subscribe(new Consumer<Object>() {
                @Override
                public void accept(Object o) throws Exception {
                    Toast.makeText(MainActivity.this, "11111111111", Toast.LENGTH_LONG).show();
                }
            });
```

