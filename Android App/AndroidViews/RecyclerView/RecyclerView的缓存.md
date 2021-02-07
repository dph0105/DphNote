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

从getScrapOrHiddenOrCachedHolderForPosition方法名，可以知道这个方法是根据position从Scrap 或者Hidden 或者 Cache中取到ViewHolder。所以该方法首先从我们正要讨论的mAttachedScrap中查询ViewHolder。当mAttachedScrap中的ViewHolder符合条件时，直接返回之~。这些条件包括：

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

```java
//in LinearLayoutManager
View next(RecyclerView.Recycler recycler) {
    if (mScrapList != null) {
        return nextViewFromScrapList();
    }
    final View view = recycler.getViewForPosition(mCurrentPosition);
    mCurrentPosition += mItemDirection;
    return view;
}
```

我们看到，再往上，就到了LayoutManager中的next方法中了，RecyclerView通过LayoutManager管理子View，在RecyclerView中，会调用 `dispatchLayout() -> dispatchLayoutStep2() -> LayoutManager.onLayoutChildren(mRecycler, mState) -> LayoutManager.fill -> LayoutManager.layoutChunk -> LayoutManager.next` 

在layoutChunk中，循环调用next函数，获取View填充入RecyclerView中。

#### 路线二:

##### 1.getScrapOrCachedViewForId

根据这个方法名，我们可以知道它是根据Id来寻找ViewHolder

```java
ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
            // Look in our attached views first
            final int count = mAttachedScrap.size();
            for (int i = count - 1; i >= 0; i--) {
                final ViewHolder holder = mAttachedScrap.get(i);
                if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
                    if (type == holder.getItemViewType()) {
                        holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                        if (holder.isRemoved()) {
                            // this might be valid in two cases:
                            // > item is removed but we are in pre-layout pass
                            // >> do nothing. return as is. make sure we don't rebind
                            // > item is removed then added to another position and we are in
                            // post layout.
                            // >> remove removed and invalid flags, add update flag to rebind
                            // because item was invisible to us and we don't know what happened in
                            // between.
                            if (!mState.isPreLayout()) {
                                holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE
                                        | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);
                            }
                        }
                        return holder;
                    } else if (!dryRun) {
                        // if we are running animations, it is actually better to keep it in scrap
                        // but this would force layout manager to lay it out which would be bad.
                        // Recycle this scrap. Type mismatch.
                        mAttachedScrap.remove(i);
                        removeDetachedView(holder.itemView, false);
                        quickRecycleScrapView(holder.itemView);
                    }
                }
            }
    		//....
        }

```

该方法也首先从mAttachedScrap中获取ViewHolder，符合`holder.getItemId() == id`以及` !holder.wasReturnedFromScrap()`的ViewHolder会被取出，并判断ViewHolder的type是否是目标Type：`type == holder.getItemViewType()`，我们看到，如果type不符合，这里会将ViewHolder从mAttachedScrap缓存中移除。

##### 2. tryGetViewHolderForPositionByDeadline

```java
        @Nullable
        ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
            // 1) Find by position from scrap/hidden list/cache
            if (holder == null) {
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                //...
            }
            if (holder == null) {
                //...
                final int type = mAdapter.getItemViewType(offsetPosition);
                // 2) Find from scrap/cache via stable ids, if exists
                if (mAdapter.hasStableIds()) {
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                            type, dryRun);
                    if (holder != null) {
                        // update position
                        holder.mPosition = offsetPosition;
                        fromScrapOrHiddenOrCache = true;
                    }
                }
                //...
          
            }
            return holder;
        }

```

getScrapOrCachedViewForId同样是tryGetViewHolderForPositionByDeadline方法，但是我们可以看到有个前提是需要设置mAdapter.hasStableIds()才会进入这一步，这里验证的是Adapter的mHasStableIds是否为 true，需要我们调用`setHasStableIds`手动设置，表示每个Item的Id固定不变。

再接下来的路线则与路线一相同

### 存入缓存

mAttachScrap通过scrapView方法增加ViewHolder

```java
        void scrapView(View view) {
            final ViewHolder holder = getChildViewHolderInt(view);
            if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
                    || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
                if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
                    throw new IllegalArgumentException("Called scrap view with an invalid view."
                            + " Invalid views cannot be reused from scrap, they should rebound from"
                            + " recycler pool." + exceptionLabel());
                }
                holder.setScrapContainer(this, false);
                mAttachedScrap.add(holder);
            } else {
                if (mChangedScrap == null) {
                    mChangedScrap = new ArrayList<ViewHolder>();
                }
                holder.setScrapContainer(this, true);
                mChangedScrap.add(holder);
            }
        }
```

scrapView通过传入的view获取到它的ViewHolder，符合条件的ViewHolder会被存入mAttachedScrap中

1. holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID) ，即ViweHolder有FLAG_REMOVED 或者FLAG_INVALID标志，表示ViewHolder的数据指向被删除的项 或者是 ViewHolder的数据已经无效
2.  !holder.isUpdated() 表示ViewHolder的view已经无效
3. canReuseUpdatedViewHolder(holder)

符合其中之一，则加入到 mAttachedScrap

然后分为两个路线：

#### 路线一

```java
private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    //如果设置了忽略viewholder的缓存
    if (viewHolder.shouldIgnore()) {
        if (DEBUG) {
            Log.d(TAG, "ignoring view " + viewHolder);
        }
        return;
    }
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        detachViewAt(index);
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}
```

这里如果viewholder的数据已经无效，并且viewholdr并没有指向被移除的数据项，并且



#### 路线三

#### 1. getScrapViewAt

```java
        View getScrapViewAt(int index) {
            return mAttachedScrap.get(index).itemView;
        }
```

getScrapViewAt方法中，直接获取mAttachedScrap中第index个ViewHolder，并取到它的itemView

#### 2. removeAndRecycleScrapInt



```java
        void removeAndRecycleScrapInt(Recycler recycler) {
            final int scrapCount = recycler.getScrapCount();
            // Loop backward, recycler might be changed by removeDetachedView()
            for (int i = scrapCount - 1; i >= 0; i--) {
                final View scrap = recycler.getScrapViewAt(i);
                final ViewHolder vh = getChildViewHolderInt(scrap);
                if (vh.shouldIgnore()) {
                    continue;
                }
                // If the scrap view is animating, we need to cancel them first. If we cancel it
                // here, ItemAnimator callback may recycle it which will cause double recycling.
                // To avoid this, we mark it as not recycleable before calling the item animator.
                // Since removeDetachedView calls a user API, a common mistake (ending animations on
                // the view) may recycle it too, so we guard it before we call user APIs.
                vh.setIsRecyclable(false);
                if (vh.isTmpDetached()) {
                    mRecyclerView.removeDetachedView(scrap, false);
                }
                if (mRecyclerView.mItemAnimator != null) {
                    mRecyclerView.mItemAnimator.endAnimation(vh);
                }
                vh.setIsRecyclable(true);
                recycler.quickRecycleScrapView(scrap);
            }
            recycler.clearScrap();
            if (scrapCount > 0) {
                mRecyclerView.invalidate();
            }
        }

```

