---
title: code1
date: 2019-07-19 18:30:29
tags:
---

## 下拉刷新分页加载控件分析

---

#### 主流下拉刷新控件横评

备注：我将从实现原理、易用性、扩展性、稳定性三个方面比较

易用性：包括 1、使用是否方便，xml java均可配置使用 2、是否将常用的逻辑功能封装（分页计算、footer等），使用者不关心细节 3、对一些常用的扩展是否已支持可配置（header的样式等）

扩展性：包括 1、支持的下拉、分页的ViewGroup是否可方便扩展 2、header footer等是否扩展方便

稳定性：包括 1、github活跃性，issue是否及时处理 2、上线后控件内部crash

##### 一、最早的先行者：XListView （[https://github.com/Maxwin-z/XListView-Android](https://github.com/Maxwin-z/XListView-Android)）

1、实现原理：

直接extends ListView，使用也和Listview一样，header和footer也是采用ListView自带的功能，仅对layout做了封装XListViewFooter和XListViewHeader。

从代码结构来看，非常简单。header和footer的显示，通过listview的onTouchEvent来判断。

[图片上传失败...(image-b71760-1541780666346)]

2、易用性：与ListView同，但是下拉和分页的可配置性几乎没有，常用封装全无

3、扩展性：很差，只能在使用ListView时使用，扩展需要改动代码，代码本身扩展性考虑很少。

4、稳定性：github已停更，线上经典crash难于解决。

作为最早Android下拉刷新功能的实践者，仅有有历史意义

##### 二、广泛应用者：PullToRefresh ([https://github.com/chrisbanes/Android-PullToRefresh](https://github.com/chrisbanes/Android-PullToRefresh))

###### 1、实现原理：

其类图可以较好的说明，其架构方式：

[图片上传失败...(image-4335ec-1541780666347)]

PullToRefresh控件基本奠定了 下拉刷新控件的架构形式：架构两大部分，

1）一部分下拉分页的骨架：核心content的加载和扩展、footer和header的加载、交互（state的分发）等

2）一部分footer和header的处理：footer header架构对不同state的处理，及自身的扩展和定制。

依据以上两部分，基于IPullToRefresh和 ILoadingLayout两个接口开发。

1. 核心骨架

    ```
        private void init(Context context, AttributeSet attrs) {
              ........//init Codes
        
              setGravity(Gravity.CENTER);
        
              ViewConfiguration config = ViewConfiguration.get(context);
              mTouchSlop = config.getScaledTouchSlop();
        
              ....//Parse styleable
        
              // Refreshable View 用于扩展
              // By passing the attrs, we can add ListView/GridView params via XML
              mRefreshableView = createRefreshableView(context, attrs);
              addRefreshableView(context, mRefreshableView);
        
              // We need to create now layouts now
          //createLoadingLayout方法构造header 和 footer
              mHeaderLayout = createLoadingLayout(context, Mode.PULL_FROM_START, a);
              mFooterLayout = createLoadingLayout(context, Mode.PULL_FROM_END, a);
        
        
              if (a.hasValue(R.styleable.PullToRefresh_ptrOverScroll)) {
                  mOverScrollEnabled = a.getBoolean(R.styleable.PullToRefresh_ptrOverScroll, true);
              }
        
              if (a.hasValue(R.styleable.PullToRefresh_ptrScrollingWhileRefreshingEnabled)) {
                  mScrollingWhileRefreshingEnabled = a.getBoolean(
                          R.styleable.PullToRefresh_ptrScrollingWhileRefreshingEnabled, false);
              }
        
              // Let the derivative classes have a go at handling attributes, then
              // recycle them...
              handleStyledAttributes(a);
              a.recycle();
        
              // Finally update the UI for the modes
          //updateUIForMode 用于添加footer和header到linearlayout中
              updateUIForMode();
          }
        
        
    
    ```

PullToRefreshBase本身是LinearLayout，其支持横向（很少用）和纵向的下拉刷新，把contentView（mRefreshableView）和footer header作为childView添加到其中。

扩展方式：

abstract方法createRefreshableView（），在子类中实现用于扩展contentView

footer header的扩展通过createLoadingLayout()返回，只要继承自LoadingLayout即可扩展。当然控件本身提供了集中常用的Loadinglayout(FlipLoadingLayout RotateLoadingLayout)

交互处理：

如何从手势的变化决定header以及footer的state呢？是通过onInterceptTouchEvent和OnTouchEvent。

和其他的touch事件处理类似，onInterceptTouchEvent方法作为前置准备，onTouchEvent方法实际处理手势操作

    ```
          @Override
          public final boolean onTouchEvent(MotionEvent event) {
        
            if (!isPullToRefreshEnabled()) {
              return false;
            }
        
            // If we're refreshing, and the flag is set. Eat the event
            if (!mScrollingWhileRefreshingEnabled && isRefreshing()) {
              return true;
            }
        
            if (event.getAction() == MotionEvent.ACTION_DOWN && event.getEdgeFlags() != 0) {
              return false;
            }
        
            switch (event.getAction()) {
              case MotionEvent.ACTION_MOVE: {
                if (mIsBeingDragged) {
                  mLastMotionY = event.getY();
                  mLastMotionX = event.getX();
                  pullEvent();//处理拉动过程中，header footer状态的变化
                  return true;
                }
                break;
              }
        
              case MotionEvent.ACTION_DOWN: {
                if (isReadyForPull()) {
                  mLastMotionY = mInitialMotionY = event.getY();
                  mLastMotionX = mInitialMotionX = event.getX();
                  return true;
                }
                break;
              }
        
              case MotionEvent.ACTION_CANCEL:
              case MotionEvent.ACTION_UP: {
                //ACTION_UP事件的处理，在不同state下松手，处理方式的不同
                if (mIsBeingDragged) {
                  mIsBeingDragged = false;
        
                  if (mState == State.RELEASE_TO_REFRESH
                      && (null != mOnRefreshListener || null != mOnRefreshListener2)) {
                    //拉动结束，在RELEASE_TO_REFRESH状态下松手，变为REFRESHING
                    setState(State.REFRESHING, true);
                    return true;
                  }
        
                  // If we're already refreshing, just scroll back to the top
                  if (isRefreshing()) {
                    //拉动结束，在REFRESHING状态下松手，回到原点
                    smoothScrollTo(0);
                    return true;
                  }
        
                  // If we haven't returned by here, then we're not in a state
                  // to pull, so just reset
                  //拉动结束，在其他状态（PULL_TO_REFRESH）下松手，reset到初始状态
                  setState(State.RESET);
        
                  return true;
                }
                break;
              }
            }
        
            return false;
          }
        
    
    ```

    ```
        /**
         * Actions a Pull Event
         *
         * @return true if the Event has been handled, false if there has been no
         *         change
         */
        private void pullEvent() {
          final int newScrollValue;
          final int itemDimension;
          final float initialMotionValue, lastMotionValue;
        
        
          switch (mCurrentMode) {
            case PULL_FROM_END:
              newScrollValue = Math.round(Math.max(initialMotionValue - lastMotionValue, 0) / FRICTION);
              itemDimension = getFooterSize();
              break;
            case PULL_FROM_START:
            default:
              newScrollValue = Math.round(Math.min(initialMotionValue - lastMotionValue, 0) / FRICTION);
              itemDimension = getHeaderSize();
              break;
          }
        
          setHeaderScroll(newScrollValue);
        
          if (newScrollValue != 0 && !isRefreshing()) {
            float scale = Math.abs(newScrollValue) / (float) itemDimension;
            switch (mCurrentMode) {
              case PULL_FROM_END://上拉分页
                mFooterLayout.onPull(scale);//根据滑动的位置更新footerLayout
                break;
              case PULL_FROM_START://下拉刷新
              default:
                mHeaderLayout.onPull(scale);//根据滑动的位置更新headerLayout
                break;
            }
        
            //根据滑动的位置（是否超过阈值），决定状态PULL_TO_REFRESH or RELEASE_TO_REFRESH
            if (mState != State.PULL_TO_REFRESH && itemDimension >= Math.abs(newScrollValue)) {
              setState(State.PULL_TO_REFRESH);
            } else if (mState == State.PULL_TO_REFRESH && itemDimension < Math.abs(newScrollValue)) {
              setState(State.RELEASE_TO_REFRESH);
            }
          }
        }
        
    
    ```

从以上代码可以看到，从手指开始滑动控件的不同状态，PullToRefreshBase的不同状态的流转，在流转的过程中，不只更新了state状态，也对footer和header进行了同步。

从以上代码容易理解下拉刷新的逻辑脉络，但是上拉分页加载是怎么实现的呢?

PullToRefreshBase控件通过mCurrentMode来区分上拉和下拉，其实上拉和下拉的逻辑，从整体上是可以归一的，有几个关键点

1、判断上拉 下拉的逻辑阈值：isReadyForPullStart（）isReadyForPullEnd（）分别是下拉 上拉的阈值方法，子类需要根据 mRefreshableView来实现

2、在不同的state下做不同的处理: 两者都有 reset PULL_TO_REFRESH RELEASE_TO_REFRESH REFRESHING等状态，可能上拉不需要区分PULL_TO_REFRESH RELEASE_TO_REFRESH两种state而已。所以既然都是基于一套state的处理方案，那么根据手势滑动方向决定当前mCurrentMode，进而交给header 或 footer来处理state就是可行的。

1. footer和header的扩展和处理

刚才说到了footer和header是在同一套state状态下的处理机制，其回调也类似。所以两者继承同一接口和基类。PullToRefreshBase控件采用了Proxy的方式，实现了二者的统一调用。

也就是说LoadingLayoutProxy 、headerLoadingLayout、footerLoadingLayout均实现ILoadingLayout，LoadingLayoutProxy是headerLoadingLayout与footerLoadingLayout二者的代理，在state的流转过程中，通过LoadingLayoutProxy的调用，达到header 和footer两个loadingLayout的同步调用。

LoadingLayout基类已经实现了基本的layout，我们自己定制的子类（例如CustomLoadingLayout）,对里面的动画，文案等进行定制即可，基于ILoadingLayout接口完全重写一个新的,目前看不行，一方面PullToRefreshBase控件内部很多地方强转到LoadingLayout。而且LoadingLayout基类（abstract类）预留了stated的回调抽象方法，供子类实现:

    ```
        protected abstract void onLoadingDrawableSet(Drawable imageDrawable);
        
        protected abstract void onPullImpl(float scaleOfLayout);
        
        protected abstract void pullToRefreshImpl();
        
        protected abstract void refreshingImpl();
        
        protected abstract void releaseToRefreshImpl();
        
        protected abstract void resetImpl();
        
    
    ```

###### 2、易用性：

实现原理说了这么多，基本上把控件的基本架构和处理流程都涉及了。一般来说，通用控件架构设计的初衷一般为易用性和扩展性考虑。好的架构能够兼顾这二者。我们具体看一下：

1、使用是否方便，xml和java代码都可以初始化和配置控件，这是控件设计初期就考虑到的

2、我们知道为了保证扩展性，架构上的实现不能过于具体，否则灵活性降低。架构上基于接口和抽象类进行设计，能保证在整体架构内部方便扩展。同时也提供了一些常用的具体实现类，比如PullToRefreshListView FlipLoadingLayout等对于一般的使用者可以省去二次开发的时间

3、一些业务上的常用逻辑：（分页计算、footer多个状态的显示等）没有集成，需要二次开发

###### 3、扩展性：

mRefreshableView的设计理念，可以说让控件理论上可以支持任何视图类（ViewGroup）的下拉刷新操作，比如后期扩展RecyclerView、ViewPager等。

从类图中可以看出 PullToRefreshBase的多层子类，设计合理，层次分明。二次开发中可以选择合适的基类进行扩展。

LoadingLayoutProxy机制的引入，为实现更多LoadingLayout的state流转提供了可能。

模板方法设计模式，基于接口开发，abstract基类，易于扩展和维护

###### 4、稳定性：

github star 8700多，多个工程中考验，类库内部崩溃率较低。

##### 三、官方控件：SwipeRefreshLayout

一两句就能说清：

这个控件作为targetView（比如listview）的parentView出现，而且SwipeRefreshLayout只能有一个childView。

交互上比较单一，materialDesign风格，loading图标在targetView之上显示，targetView本身可以是任何view。

##### 四、二次开发的控件：LRecyclerView

LRecyclerView是csdn大牛‘一叶飘舟’所著，设计的初衷是为了打造一个更为好用的RecyclerView,一切基于RecyclerView架构搭建。增加了header footer功能（不同于listview，为了扩展性，原生的RecyclerView并不支持header和footer）。增加了下拉刷新和上拉分页加载功能（这个功能后来被更广泛使用，所以在已有架构上支持了PullScrollView、PullWebView）。最终达到了现有的面貌。

目前我们已经将RecyclerView作为开发的主力控件，那么基于RecyclerView的一个易用性、扩展性和稳定性逗号的控件，就是我们研究的目标。

###### 1、实现原理：

有了以上的背景，我们对LRecyclerView这个控件会有一个大概认识。我们看下代码分布：

[图片上传失败...(image-ea4ae6-1541780666347)]

从他的代码分布可以看出，基本是围绕LRecyclerview开展的。类之间的相互关系比较简单，就不用类图展开了。

LRecyclerView是主体类，核心代码在LRecyclerView中。（另外LuRecyclerView是为了适应google官方控件SwipeRefreshLayout而改造的，因为直接使用SwipeRefreshLayout嵌套LRecyclerView会有事件冲突。在分析上可以暂时略过）。

以下我们将从两个方面分析 1、LRecyclerView是如何在RecyclerView基础上加上footer和header；2、LRecyclerView是如何实现下拉刷新和上拉分页加载的。

1. LRecyclerView是如何在RecyclerView基础上加上footer和header的

我们知道listview原生支持footer和header，如果我们看过listview的源码的话，就知道他们是在通过adapter实现的，listView在添加header时代码如下：

    ```
        public void addHeaderView(View v, Object data, boolean isSelectable) {
        
          if (mAdapter != null) {
            //如果是设置header，那么通过HeaderViewListAdapter的代理wrapperadapter来包装真正的adapter
              if (!(mAdapter instanceof HeaderViewListAdapter)) {
                  wrapHeaderListAdapterInternal();
              }
        
              // In the case of re-adding a header view, or adding one later on,
              // we need to notify the observer.
              if (mDataSetObserver != null) {
                  mDataSetObserver.onChanged();
              }
          }
        }
        
    
    ```

当添加header时，将mAdapter通过方法wrapHeaderListAdapterInternal()包装，HeaderViewListAdapter是mAdapter的代理类，可以看到类内部有成员变量mAdapter,就是ListView的使用者真实创建的adapter。

通过以下代码我们就一目了然他的实现原理了：实现原理请参考注释。

    ```
        public View getView(int position, View convertView, ViewGroup parent) {
            // Header (negative positions will throw an IndexOutOfBoundsException)
            int numHeaders = getHeadersCount();
            //如果是position指向header，那么从mHeaderViewInfos返回对应view
            if (position < numHeaders) {
                return mHeaderViewInfos.get(position).view;
            }
        
            // Adapter
            final int adjPosition = position - numHeaders;
            int adapterCount = 0;
            if (mAdapter != null) {
                adapterCount = mAdapter.getCount();
                //如果是position指向mAdapter实际列表数据，那么调用mAdapter.getView
                if (adjPosition < adapterCount) {
                    return mAdapter.getView(adjPosition, convertView, parent);
                }
            }
        
            //如果是position指向footer，那么从mFooterViewInfos返回对应view
            // Footer (off-limits positions will throw an IndexOutOfBoundsException)
            return mFooterViewInfos.get(adjPosition - adapterCount).view;
        }
        
    
    ```

同时getCount getItemType getItem等实现均对 footer和header进行了考虑，这样包装类封装了mAdapter本身和 footer header，将他们作为一个整体提供给listview。

本控件的作者借鉴了这个思路，设计了代理类LRecyclerViewAdapter，类里类似的也含有mInnerAdapter实际的adapter，mHeaderViews和mFooterViews则用于保存信息。

    ```
        @Override
        public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
            //分别RefreshHeader header footer三种类型返回不同的ViewHolder
            //这里RefreshHeader没有像PullRefreshView一样作为listview之外的view存在，而是放入
            //adapter内部让listview(RecyclerView)一起加载。
            //如何虽手势控制RefreshHeader的Layout，后面详细说。
            if (viewType == TYPE_REFRESH_HEADER) {
                return new ViewHolder(mRefreshHeader.getHeaderView());
            } else if (isHeaderType(viewType)) {
                return new ViewHolder(getHeaderViewByType(viewType));
            } else if (viewType == TYPE_FOOTER_VIEW) {
                return new ViewHolder(mFooterViews.get(0));
            }
            return mInnerAdapter.onCreateViewHolder(parent, viewType);
        }
        
    
    ```

和listview的HeaderViewListAdapter一样，LRecyclerViewAdapter也是类似的处理：

    ```
        @Override
        public int getItemCount() {
            if (mInnerAdapter != null) {
                //此处+1，是考虑到RefreshHeader，就是说header和RefreshHeader是不同的功能，可能同时出现
                //而footer作为一般的footer或者上拉加载的footer，只会出现一种
                return getHeaderViewsCount() + getFooterViewsCount() + mInnerAdapter.getItemCount() + 1;
            } else {
                return getHeaderViewsCount() + getFooterViewsCount() + 1;
            }
        }
        
    
    ```

在阅读以上代码时，大家不免会有个疑问，LRecyclerView的使用上并不像listview那样简练，LRecyclerView在设置adapter时，需要手动创建innerAdapter和wrapperadapter，将innerAdapter包裹进WrapperAdapter后设置给LRecyclerView；而listview会根据header/footer使用情况自动创建wrapperadapter,使用者并不知道代理类的存在。此处的设计在文章的最后会阐述我的一些看法。

1. LRecyclerView是如何实现下拉刷新和上拉分页的

如何下拉刷新：LRecyclerView下拉刷新也是是通过onInterceptTouchEvent和onTouchEvent来实现的，具体的实现和PullRefreshView类似，此处不单独分析了。通过接口IRefreshHeader来控制RefreshHeader的状态改变。刷新后通过OnRefreshListener接口通知业务刷新数据。

如何分页加载：利用RecyclerView的onScrolled回调，控件滑动过程中不断回调此方法，通过判断是否滑动到最底部来决定是否上拉加载，代码如下：

    ```
        if (mLoadMoreListener != null && mLoadMoreEnabled) {
            int visibleItemCount = layoutManager.getChildCount();
            int totalItemCount = layoutManager.getItemCount();
            if (visibleItemCount > 0
                    && lastVisibleItemPosition >= totalItemCount - 1
                    && totalItemCount > visibleItemCount
                    && !isNoMore
                    && !mRefreshing) {
        
                mFootView.setVisibility(View.VISIBLE);
                if (!mLoadingData) {
                    mLoadingData = true;
                    //更新footerView的状态
                    mLoadMoreFooter.onLoading();
                    if (mWrapAdapter != null) {
                        //回调业务 分页加载更多
                        mWrapAdapter.loadMore(mLoadMoreListener);
                    }
                }
            }
        
        }
        
    
    ```

###### 2、易用性：

此控件通过将IRefreshHeader和ILoadMoreFooter两个接口的拆分，相比较PullRefreshView对于上拉footer的处理更加直接和便捷。ILoadMoreFooter的不同接口更加适应于分页加载的不同状态。并且不同状态的文案是可以定制的：```
    public void setFooterViewHint(String loading, String noMore, String noNetWork)

```这样对于上拉分页的情况，不需要业务再对控件做二次开发（PullRefreshView需要），是更加易用的。

但是业务上对于分页加载需求的逻辑负担还是比较大，集中在以下两点（PullRefreshView也存在此问题）

1）分页pageNumber pageSize等需要业务维护，而这些逻辑都是通用的。

2）判断是否需要加载更多，还是没有更多数据，的逻辑业务需要维护，也是可以通用的。

这两个问题其实都可以在wrapperAdapter中通过统一的逻辑来处理，只不过业务加载后要通知控件：除了LRecyclerView在加载更多时通知业务onLoadMore，业务在加载更多后也要通过接口ILoadCallback把结果传入wrapperAdapter中，这样wrapperAdapter便可以在回调后根据当前adapter内的数据统一处理pageNumer等字段，维护是否加载更多的状态了。

我们自定义的ILoadCallback接口，业务在onLoadMore处理完后，要根据返回的结果调用的接口。

    ```
        public interface ILoadCallback {
            //业务loadMore的结果 success和failue都通知wrapperAdapter
            //wrapperAdapter通过innerAdapter的数据就可以处理了
            void onSuccess();
        
            void onFailure();
        }
        
    
    ```

WrapperAdapter对接口调用的处理：维护pageNumber,和footer是否加载更多等状态

    ```
        private ILoadCallback mLoadCallback = new ILoadCallback() {
          @Override
          public void onSuccess() {
              notifyDataSetChanged();
              if ((mInnerAdapter.getItemCount() % getItemNumInPage()) == 0){
                //判断还需要加载下一页
                  mCurrentPage++;
                  if (mLRecyclerView != null) {
                      mLRecyclerView.setNoMore(false);
                  }
              } else {
                //判断没有更多数据，并将footerview设置为noMore
                  if (mLRecyclerView != null) {
                      mLRecyclerView.setNoMore(true);
                  }
              }
              if (mLRecyclerView != null) {
                  mLRecyclerView.refreshComplete(getItemNumInPage());
              }
          }
        
          @Override
          public void onFailure() {
            //失败时统一提示，并集成再次点击，多加载一次的功能
              mLRecyclerView.refreshComplete(getItemNumInPage());
              mLRecyclerView.setOnNetWorkErrorListener(new OnNetWorkErrorListener() {
                  @Override
                  public void reload() {
                      if (mLoadMoreCallback != null) {
                          mLoadMoreCallback.onLoadMore(mCurrentPage, getItemNumInPage(), mLoadCallback);
                      }
                  }
              });
          }
        };
        
    
    ```

经过这样进一步的封装，LRecyclerView的使用易用性进一步提升了。可以说比PullRefreshView本身的易用性要强一些，尤其是在分页加载的逻辑封装方面

###### 3、扩展性：

PullRefreshView自身支持所有ViewGroup的下拉刷新。我觉得LRecyclerView与PullRefreshView相比，在架构上牺牲了一些扩展性，但易用性有很大的提升，应用场景有较强的针对性。而扩展性方面，利用Recyclerview自身很强的扩展性，就可以应付大部分使用场景。当然header footer RefreshHeader这些样式的扩展是自然支持的。

###### 4、稳定性：

github star数在2000以上，issue修改及时，在二次开发的过程中，上拉分页的footer状态维护有些小bug，但是基本不影响稳定性，产品上线后控件的崩溃率一直很低。基本可以放心使用。

###### 5、其他的思考：

wrapperAdapter的设置：

上文中提及过的，WrapperAdapter和innerAdapter都需要在业务上的新建有点鸡肋（因为可以在LRecyclerView setAdatper时，内部创建wrapperAdapter，和listview的做法一致），作者这么做的原因，我想可能是WrapperAdapter承载了很多框架业务的功能，那么业务持有此变量可以非常方便的调用WrapperAdapter的接口。在我看来，较为合理的方式还是将WrapperAdapter不对外暴露，将原来WrapperAdapter的对外接口改到LRecyclerView来实现。这样用户调用方便，同时对控件的封装性更好。

此封装方案我在demo project中试验过，没有太大问题，可能有些细节需要处理，后续我们的控件二次开发会采用这种方式。


