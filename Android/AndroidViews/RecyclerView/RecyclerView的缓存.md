RecyclerView通过它的子类Recycler来管理内部View的缓存策略。大家通常对RecyclerView的缓存定为四级，分别对应Recycler中的四个对象：

- ArrayList<ViewHolder> mAttachedScrap 和ArrayList<ViewHolder> mChangedScrap
- ArrayList<ViewHolder> mCachedViews
- ViewCacheExtension mViewCacheExtension
- RecycledViewPool mRecyclerPool



除了ViewCacheExtension 是交给用户自定义的配置的之外，我们来看其他三个缓存是如何工作的。

首先是mAttachedScrap

mAttachedScrap中缓存的ViewHolder是什么时候取出来的？

## mAttachedScrap

### 取出缓存

取出mAttachedScrap中的ViewHolder有几个方法，我们顺着方法来看调用路线：

#### 路线一：

##### 1. getScrapOrHiddenOrCachedHolderForPosition

```java
//in RecyclerView.Recycler
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
     final int scrapCount = mAttachedScrap.size();
    // Try first for an exact, non-invalid match from scrap.
     for (int i = 0; i < scrapCount; i++) {
         final ViewHolder holder = mAttachedScrap.get(i);//！！！！！！！
         if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                 && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
                holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                return holder;
         }
     }
    ...
}
```

从getScrapOrHiddenOrCachedHolderForPosition方法名，可以知道这个方法是从Scrap 或者Hidden 或者 Cache中取到ViewHolder。所以该方法首先从我们正要讨论的mAttachedScrap中查询ViewHolder。当mAttachedScrap中的ViewHolder符合条件时，直接返回之~。这些条件包括：

1. **!holder.wasReturnedFromScrap()**，即 ViewHolder的Flag中**没有FLAG_RETURNED_FROM_SCRAP**，表示它并不是一个已经从Scrap中取出的即将被addView的View的ViewHolder
2. **holder.getLayoutPosition() == position**，即该ViewHolder最近一次的布局中的position与现在要取的position相同
3. **!holder.isInvalid()**，即ViewHolder的Flag中**没有FLAG_INVALID**，表示该ViewHolder上的数据还有效
4. **mState.mInPreLayout || !holder.isRemoved()**，即RecyclerView处于预布局状态，或者该ViewHolder的Flag中**没有FLAG_REMOVED**，表示该ViewHolder不是属于要被删除的数据项的ViewHolder

当符合条件后，给取到的ViewHolder添加标志位FLAG_RETURNED_FROM_SCRAP，表示它重新从mAttachedScrap取出，并即将被addView到RecyclerView中

##### 2.  tryGetViewHolderForPositionByDeadline

```java
//in Recycler   
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
      		ViewHolder holder = null;
            // 0) If there is a changed scrap, try to find from there
            //...
            // 1) Find by position from scrap/hidden list/cache
            if (holder == null) {
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);//！！！！
                if (holder != null) {
                    if (!validateViewHolderForOffsetPosition(holder)) {
                        // recycle holder (and unscrap if relevant) since it can't be used
                        if (!dryRun) {
                            // we would like to recycle this but need to make sure it is not used by
                            // animation logic etc.
                            holder.addFlags(ViewHolder.FLAG_INVALID);
                            if (holder.isScrap()) {
                                removeDetachedView(holder.itemView, false);
                                holder.unScrap();
                            } else if (holder.wasReturnedFromScrap()) {
                                holder.clearReturnedFromScrapFlag();
                            }
                            recycleViewHolderInternal(holder);
                        }
                        holder = null;
                    } else {
                        fromScrapOrHiddenOrCache = true;
                    }
                }
            }
  }
```

tryGetViewHolderForPositionByDeadline 也会从Scrap 或者Hidden 或者 Cache中取到ViewHolder，当取不到时，就创建一个。

当这里从上一步中成功取到一个ViewHolder时，会验证这个ViewHolder。这里主要验证ViewHolder的状态以及position等，验证失败的后果会怎么样，我们后面再讲。

##### 3. getViewForPosition

```java
//in Recycler   
public View getViewForPosition(int position) {
            return getViewForPosition(position, false);
        }
		
        View getViewForPosition(int position, boolean dryRun) {
            return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
        }

```

getViewForPosition直接就调用了tryGetViewHolderForPositionByDeadline，并且这里的dryRun只为false，因为getViewForPosition(int position, boolean dryRun)并没有其他地方使用到了。

##### 4. next

```
View next(RecyclerView.Recycler recycler) {
    if (mScrapList != null) {
        return nextViewFromScrapList();
    }
    final View view = recycler.getViewForPosition(mCurrentPosition);
    mCurrentPosition += mItemDirection;
    return view;
}
```



