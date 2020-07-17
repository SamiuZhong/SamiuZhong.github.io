```xml
<androidx.appcompat.widget.Toolbar
    android:id="@+id/toolbar"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:minHeight="?attr/actionBarSize"
    app:contentInsetStartWithNavigation="0dp"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintHorizontal_bias="0.0"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent"
    app:marginTopSystemWindowInsets="@{true}"
    app:menu="@menu/learn"
    app:navigationIcon="@drawable/ic_back">
    
  <ImageView
      android:layout_width="24dp"
      android:layout_height="24dp"
      app:srcCompat="@drawable/ic_logo" />
</androidx.appcompat.widget.Toolbar>
```

