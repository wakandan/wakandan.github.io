---
layout: post
title: "Basic operations for opencv beginner"
---

# Basic importing

```
import numpy as np
import cv2
from matplotlib import pyplot as plt
```

# Loading an image

```
img = cv2.imread(file_name, 0)
```

# Padding an image

```
cv2.copyMakeBorder(img, <top>, <bottom>, <left>, <right>, CV2.BORDER_REPLICATE)
```

# Showing an image

```
plt.imshow(img, cmap='gray', intepolation='bicubic')
plt.show()
```
