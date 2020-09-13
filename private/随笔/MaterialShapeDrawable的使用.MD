```kotlin
private val backgroundShapeDrawable: MaterialShapeDrawable by lazy(NONE) {
    val backgroundContext = binding.backgroundContainer.context
    MaterialShapeDrawable(
        backgroundContext,
        null,
        R.attr.bottomSheetStyle,
        0
    ).apply {
        fillColor = ColorStateList.valueOf(
            backgroundContext.themeColor(R.attr.colorPrimarySurfaceVariant)
        )
        elevation = resources.getDimension(R.dimen.plane_08)
        initializeElevationOverlay(requireContext())
    }
}

private val foregroundShapeDrawable: MaterialShapeDrawable by lazy(NONE) {
    val foregroundContext = binding.foregroundContainer.context
    MaterialShapeDrawable(
        foregroundContext,
        null,
        R.attr.bottomSheetStyle,
        0
    ).apply {
        fillColor = ColorStateList.valueOf(
            foregroundContext.themeColor(R.attr.colorPrimarySurface)
        )
        elevation = resources.getDimension(R.dimen.plane_16)
        shadowCompatibilityMode = MaterialShapeDrawable.SHADOW_COMPAT_MODE_NEVER
        initializeElevationOverlay(requireContext())
        shapeAppearanceModel = shapeAppearanceModel.toBuilder()
            .setTopEdge(
                SemiCircleEdgeCutoutTreatment(
                    resources.getDimension(R.dimen.grid_1),
                    resources.getDimension(R.dimen.grid_3),
                    0F,
                    resources.getDimension(R.dimen.navigation_drawer_profile_image_size_padded)
                )
            )
            .build()
    }
}
```

