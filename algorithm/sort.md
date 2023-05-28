## 快速排序【andy故意搞的代码冲突】

**代码**

```java
public static void quickSort(final int[] arr){
        quickSort(arr,0,arr.length-1);
    }

    public static void quickSort(final int[]arr,int l,int r){
        if (l >= r){
            return;
        }
        int x = arr[l + r >> 1];
        int i = l - 1;
        int j = r + 1;
        while (i < j){
            while (arr[++i] < x){}
            while (arr[--j] > x){}
            if (i < j){
                int t = arr[i];
                arr[i] = arr[j];
                arr[j] = t;
            }
        }
        quickSort(arr,l,j);
        quickSort(arr,j+1,r);
    }
```

## 归并排序

**代码**

```java
public static void mergeSort(final int[] arr,int l,int r){
        if (l >= r){
            return;
        }
        int mid = l + r >> 1;
        mergeSort(arr,l,mid);
        mergeSort(arr,mid+1,r);
        int i = l;
        int j = mid + 1;
        List<Integer> temp = new ArrayList<>();
        while (i <= l && j <= r){
            if (arr[i] <= arr[j]){
                temp.add(arr[i++]);
            }else {
                temp.add(arr[j++]);
            }
        }
        while (i <= mid){
            temp.add(arr[i++]);
        }
        while (j <= r){
            temp.add(arr[j++]);
        }
        int index = 0;
        for (int k = l; k <= r ; k++) {
            arr[k] = temp.get(index++);
        }
    }
```

## 快速排序求K小

**代码**

```java
public static int minK(final int[] arr,int l,int r,int k){
        if (l >= r){
            return arr[l-1];
        }
        int i = l - 1;
        int j = r + 1;
        int p = arr[l + r >> 1];
        while (i < j){
            while (arr[++i] < p){}
            while (arr[--j] > p){}
            if (i < j){
                int t = arr[i];
                arr[i] = arr[j];
                arr[j] = t;
            }
        }
        if (j == k-1){
            return arr[j];
        }
        int sorted = j - l + 1;
        if (sorted > k){
            return minK(arr,l,j,k);
        }else {
            return minK(arr,j+1,r,k - sorted);
        }
    }
```

