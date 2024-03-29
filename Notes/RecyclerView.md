---
RecyclerView
---
#### 目录
1. 基本使用
   1. 复杂布局实现
   2. 下拉刷新&上拉加载
   3. LayoutManager
   4. ItemTouchHelper
   5. 万能 Adapter
2. RecyclerView运行机制
3. RecyclerView缓存机制
4. RecyclerView优化
##### 基本使用
###### 复杂布局实现
其实就是多个 ItemType 的场景，实现起来也很简单。定义多个 ItemTpye 和 ViewHolder，在 onCreateViewHolder 中通过 itemType 返回不同的 ViewHolder，onBindViewHolder 时根据 ViewHolder 的不同在设置不同的数据，完事。

这里想说一下头布局和尾布局的实现方式，其实也可以用上面的方式解决，但是我们可以用一种更加优雅的方式解决，那就是使用装饰者模式来实现扩展。
```
public class BaseRvAdapterWrapper extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private static final int TYPE_HEADER = 0;
    private static final int TYPE_NORMAL = 1;
    private static final int TYPE_FOOTER = 2;
    private BaseRvAdapter mBaseRvAdapter;
    private View mHeaderView;
    private View mFooterView;

    public BaseRvAdapterWrapper(BaseRvAdapter baseRvAdapter) {
        mBaseRvAdapter = baseRvAdapter;
    }

    public void setHeaderView(View headerView) {
        mHeaderView = headerView;
    }

    public void setFooterView(View footerView) {
        mFooterView = footerView;
    }

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
        if (i == TYPE_HEADER) {
            return new RecyclerView.ViewHolder(mHeaderView) {
            };
        } else if (i == TYPE_NORMAL) {
            return mBaseRvAdapter.onCreateViewHolder(viewGroup, i);
        } else {
            return new RecyclerView.ViewHolder(mFooterView) {
            };
        }
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder viewHolder, int i) {
        if (i == 0 || i == mBaseRvAdapter.getItemCount() + 1) {
            return;
        } else {
            mBaseRvAdapter.onBindViewHolder((BaseRvAdapter.MyViewHolder) viewHolder, i - 1);
        }
    }

    @Override
    public int getItemCount() {
        return mBaseRvAdapter.getItemCount() + 2;
    }

    @Override
    public int getItemViewType(int position) {
        if (position == 0) {
            return TYPE_HEADER;
        } else if (position == mBaseRvAdapter.getItemCount() + 1) {
            return TYPE_FOOTER;
        } else {
            return TYPE_NORMAL;
        }
    }
}
```
BaseRvAdapter 就是我们平常写的最基本的 Adapter，ItemView 都一样的时候。
###### 下拉刷新&上拉加载
下拉刷新直接用 SwipeRefreshLayout 就行了。

上拉加载即：
```
mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(@NonNull RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                LinearLayoutManager manager = (LinearLayoutManager) recyclerView.getLayoutManager();
                // 当不滑动时
                if (newState == RecyclerView.SCROLL_STATE_IDLE) {
                    //获取最后一个完全显示的itemPosition
                    int lastItemPosition = manager.findLastCompletelyVisibleItemPosition();
                    int itemCount = manager.getItemCount();

                    // 判断是否滑动到了最后一个item，并且是向上滑动
                    if (lastItemPosition == (itemCount - 1) && isSlidingUpward) {
                        //加载更多
                        onLoadMore();
                    }
                }
            }

            @Override
            public void onScrolled(@NonNull RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                // 大于0表示正在向上滑动，小于等于0表示停止或向下滑动
                isSlidingUpward = dy > 0;
            }
        });
```
[参考](https://blog.csdn.net/weixin_37730482/article/details/72846693?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522171167556116800222893012%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=171167556116800222893012&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-4-72846693-null-null.142^v100^pc_search_result_base4&utm_term=RecyclerView%E5%AE%9E%E7%8E%B0%E5%8A%A0%E8%BD%BD%E6%9B%B4%E5%A4%9A&spm=1018.2226.3001.4187)
###### LayoutManager
常见实现类（LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutManager），下面列出 LayoutManager 常用的 API：
```
canScrollHorizontally();//能否横向滚动
canScrollVertically();//能否纵向滚动
scrollToPosition(int position);//滚动到指定位置

setOrientation(int orientation);//设置滚动的方向
getOrientation();//获取滚动方向

findViewByPosition(int position);//获取指定位置的Item View
findFirstCompletelyVisibleItemPosition();//获取第一个完全可见的Item位置
findFirstVisibleItemPosition();//获取第一个可见Item的位置
findLastCompletelyVisibleItemPosition();//获取最后一个完全可见的Item位置
findLastVisibleItemPosition();//获取最后一个可见Item的位置
```
###### ItemTouchHelper
ItemTouchHelper 能用来实现 RecyclerView Item 的上下拖拽以及滑动删除功能，实现起来可以说是究极简单。

分为两步：

1. 实现 ItemTouchHelper.Callback 回调
2. 把 ItemTouchHelper 绑定到 RecyclerView 上
   
第一步:
```
public class DefaultItemTouchHelper<T> extends ItemTouchHelper.Callback {

    private BaseRvAdapterWrapper mBaseRvAdapterWrapper;
    private List<T> mList;

    public DefaultItemTouchHelper(BaseRvAdapterWrapper baseRvAdapterWrapper, List<T> list) {
        mBaseRvAdapterWrapper = baseRvAdapterWrapper;
        mList = list;
    }

    /**
     * 设置支持滑动、拖拽的方向
     */
    @Override
    public int getMovementFlags(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder) {
        int dragFlag = ItemTouchHelper.UP | ItemTouchHelper.DOWN;   //上下拖拽
        int swipeFlag = ItemTouchHelper.START | ItemTouchHelper.END;    //左右滑动
        return makeMovementFlags(dragFlag, swipeFlag);
    }

    /**
     * 拖拽时回调
     */
    @Override
    public boolean onMove(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder, @NonNull RecyclerView.ViewHolder viewHolder1) {
        int form = viewHolder.getAdapterPosition();
        int to = viewHolder1.getAdapterPosition();
        Collections.swap(mList, form, to);
        mBaseRvAdapterWrapper.notifyItemMoved(form, to);
        return true;
    }

    /**
     * 滑动时回调
     */
    @Override
    public void onSwiped(@NonNull RecyclerView.ViewHolder viewHolder, int i) {
        int position = viewHolder.getAdapterPosition();
        mList.remove(position);
        mBaseRvAdapterWrapper.notifyItemRemoved(position);
    }

    /**
     * 状态改变时回调
     */
    @Override
    public void onSelectedChanged(@Nullable RecyclerView.ViewHolder viewHolder, int actionState) {
        super.onSelectedChanged(viewHolder, actionState);
        if (actionState != ItemTouchHelper.ACTION_STATE_IDLE) {
            viewHolder.itemView.setBackgroundColor(Color.parseColor("#ffffff")); //设置拖拽和侧滑时的背景色
        }
    }

    /**
     * 拖拽或滑动完成之后回调
     */
    @Override
    public void clearView(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder) {
        super.clearView(recyclerView, viewHolder);
        viewHolder.itemView.setBackgroundColor(Color.parseColor("#FFFFFF"));
    }
    
    /**
     * 如果想自定义动画，可以重写这个方法
     * 根据偏移量来设置
     */
    @Override
    public void onChildDraw(@NonNull Canvas c, @NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder, float dX, float dY, int actionState, boolean isCurrentlyActive) {
        super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive);
    }
}
```
第二步：
```
ItemTouchHelper itemTouchHelper = new ItemTouchHelper(new DefaultItemTouchHelper<>(mBaseRvAdapterWrapper, mData));
itemTouchHelper.attachToRecyclerView(mRecyclerView);
```
###### 万能Adapter
```
public abstract class QuickAdapter<T> extends RecyclerView.Adapter<QuickAdapter.VH> {

    private List<T> mDatas;

    public QuickAdapter(List<T> datas) {
        mDatas = datas;
    }

    public abstract int getLayoutId(int viewType);

    public abstract void convert(VH viewHolder, T data, int position);

    @NonNull
    @Override
    public VH onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
        return VH.get(viewGroup, getLayoutId(i));
    }

    @Override
    public void onBindViewHolder(@NonNull VH vh, int i) {
        convert(vh, mDatas.get(i), i);
    }

    @Override
    public int getItemCount() {
        return mDatas.size();
    }

    static class VH extends RecyclerView.ViewHolder {

        private SparseArray<View> mViews;
        private View mConvertView;

        public VH(@NonNull View itemView) {
            super(itemView);
            mConvertView = itemView;
            mViews = new SparseArray<>();
        }

        public static VH get(ViewGroup parent, int layoutId) {
            View convertView = LayoutInflater.from(parent.getContext()).inflate(layoutId, parent, false);
            return new VH(convertView);
        }

        public <T extends View> T getView(int id) {
            View view = mViews.get(id);
            if (view == null) {
                view = mConvertView.findViewById(id);
                mViews.put(id, view);
            }
            return (T) view;
        }

        public void setText(int id, String value) {
            TextView view = getView(id);
            view.setText(value);
        }
    }
}
```
##### RecyclerView运行机制
RecyclerView 要展示 itemView，会找 LayoutManager 要 itemView，但 LayoutManager 没有 itemView，它就会找 Recycler 要 itemView，刚开始时 Recycler 也没有 View，所以 Recycler 就会找 Adapter 获取，也就是我们创建一个 Adapter 时要实现的 onCreateViewHolder()；Recycler 从 Adapter 获取 ViewHolder 也就能获取到 itemView， LayoutManager 对 View 进行测量后就能知道 RecyclerView 是否需要再显示更多的 itemView。
![image](https://github.com/Coopergp/Android/assets/163702335/ef68f024-dfb7-4419-9b1d-2155304c0d98)

##### RecyclerView缓存机制
RecyclerView 是以 ViewHolder 单位来进行回收，Recycler 是 RecyclerView 回收机制的实现类，它实现了四级缓存：

1. mAttachedScrap

   缓存在屏幕上的 ViewHolder。

2. mCachedViews

   缓存屏幕外的 ViewHolder，默认为两个。ListView 对于屏幕外的缓存都会调用 getView。

3. mViewCacheExtensions

   需要用户定制，默认不实现。

4. mRecyclerPool

   缓存池，多个 RecyclerView 共用。

   RecyclerView#getViewForPosition 方法：
   ```
    View getViewForPosition(int position, boolean dryRun) {
            return this.tryGetViewHolderForPositionByDeadline(position, dryRun, 9223372036854775807L).itemView;
        }

        @Nullable
        RecyclerView.ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
            if (position >= 0 && position < RecyclerView.this.mState.getItemCount()) {
                //...
                if (holder == null) {
                    //...
                    type = RecyclerView.this.mAdapter.getItemViewType(offsetPosition);
                    if (RecyclerView.this.mAdapter.hasStableIds()) {
                        holder = this.getScrapOrCachedViewForId(RecyclerView.this.mAdapter.getItemId(offsetPosition), type, dryRun);
                        //...
                    }

                    if (holder == null && this.mViewCacheExtension != null) {
                        View view = this.mViewCacheExtension.getViewForPositionAndType(this, position, type);
                        //...
                    }

                    if (holder == null) {
                        holder = this.getRecycledViewPool().getRecycledView(type);
                        //...
                    }

                    if (holder == null) {
						//...
                        holder = RecyclerView.this.mAdapter.createViewHolder(RecyclerView.this, type);
                       //...
                    }
                }
                }
                return holder;
            } else {
                throw new IndexOutOfBoundsException("Invalid item position " + position + "(" + position + "). Item count:" + RecyclerView.this.mState.getItemCount() + RecyclerView.this.exceptionLabel());
            }
        }
   ```
   从上诉实现可以看出，依次从 mAttachedScrap、mCachedView、mViewCacheExtension、mRecyclerPool 寻找可复用的 ViewHolder，如果是从 mAttachedScrap 或 mCachedViews 中获取的 ViewHolder，则不会调用 onBindViewHolder，而如果从 mViewCacheExtension 或 mRecyclePool 中获取的 ViewHolder，则会调用 onBindViewHolder。如果上述没拿到缓存的 ViewHolder，则会通过 createViewHolder 来创建。
##### RecyclerView优化
1. 布局优化，尽量少的布局嵌套，尽量少的控件
2. 如果RecyclerView条目高度固定，使用setHasFixedSize(true),避免多次测量条目高度
3. 小范围修改可以试试adapter.notifyItemChanged(position)或者adapter.notifyItemRangeChanged(positionStart,itemcount)
4. 如果不需要动画，就把条目显示动画取消setSupportsChangeAnimations(false)
5. 滚动状态停止加载图片，只有当处于停止状态才去加载图片。
6. Glide去加载图片
7. 在ViewHolder中设置点击事件而不是在onBindViewHolder

