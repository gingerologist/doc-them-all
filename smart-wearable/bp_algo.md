```
PS C:\nRF5_SDK\external\blood_pressure_c\src> grep -rn malloc
circular_array.c:7:  array->data = malloc(capacity * elem_size);
dynamic_array.c:13:  array->data = malloc(elem_size * capacity);
dynamic_array.c:18:  void *new_data = malloc(array->elem_size * capacity);
iir_filter.c:14:  filter->zit = (F64 *)malloc((n - 1) * sizeof(F64));
matrix.c:13:  A->data = malloc(sizeof(F64) * m * n);
peak_finding.c:20:  pf->lms = (I32 *)malloc(sizeof(I32) * (pf->max_scale / pf->downsample + 1));
peak_finding.c:24:  pf->lms2 = (I32 *)malloc(sizeof(I32) * (pf->max_cache / pf->downsample));   
PS C:\nRF5_SDK\external\blood_pressure_c\src
```



Each `CFeatureExtractor` object (fe) has 3 `CCircularArray` and 6 `CDynamicArray` objects. All initialization and memory allocation are executed in `CFeatureExtractorInit`. 



Each `CCircularArray` malloc `capacity * elem_size` bytes, in which the former is 16 bytes (padded _IVPair) and the latter is `max_peaks`. All 3 circular array use the same `max_peaks`, so the total size is 3 * 16 * `max_peaks`.



For dynamic array, the fixed cost is 6 * sizeof I32 (4 bytes) or F64 (8 bytes) respectively, for peaks and valleys. The temp dynamic array consumes `max_peaks * 8` bytes.

 













