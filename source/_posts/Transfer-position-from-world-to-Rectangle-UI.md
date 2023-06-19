---
title: 一种在Unity中从世界坐标转UI坐标的方法
date: 2021-06-19 10:21:14
tags:
- Unity
categories:
- Tech
---



世界坐标→屏幕坐标→UI坐标

```c#
Vector3 screenPos = (Camera)mainCamera.WorldToScreenPoint(worldPos);
if (RectTransformUtility.ScreenPointToLocalPointInRectangle(
    (RectTransform)uiWindow.transform, screenPos, (Camera)uiCamera, out Vector2 localPoint)) {
    transform.localPosition = localPoint + extraOffset + verticalOffset;
}
```
