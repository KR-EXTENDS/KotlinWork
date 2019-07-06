# LiveDataまとめ

~~~kotlin
    /**
     * LiveData監視
     */
    private fun subscribe() {
        val latLngObserver: Observer<LatLng> = Observer { t ->
            // マーカーの位置を更新
            mMarker?.remove()
            mMarker = mMap.addMarker(markerInit(t))
        }
        val locationObserver: Observer<Location> = Observer { t ->
            // 現在地のサークル更新
            mCircle?.remove()
            mCircle = mMap.addCircle(circleInit(t))
        }
        val rootObserver: Observer<MutableList<LatLng>> = Observer { t ->
            // ルート情報更新
            mPolyline?.remove()
            mPolyline = mMap.addPolyline(PolylineOptions().apply {
                addAll(t)
            })
            mProgressDialog?.dismiss()
        }
        mMapsViewModel.mMarkerLatLng.observe(this, latLngObserver)
        mMapsViewModel.mLocation.observe(this, locationObserver)
        mMapsViewModel.mRootInfo.observe(this, rootObserver)
    }
~~~

## Observerとは？
とても単純、値が変わるとonChangedで通知(なぜLiveDataObserverとかにしなかったのかな...)
~~~java
    /**
    * A simple callback that can receive from {@link LiveData}.
    *
    * @param <T> The type of the parameter
    *
    * @see LiveData LiveData - for a usage description.
    */
    public interface Observer<T> {
        /**
        * Called when the data is changed.
        * @param t  The new data
        */
        void onChanged(T t);
    }
~~~

## mMapsViewModel.mMarkerLatLng.observe(this, latLngObserver)
監視開始って感じ？<br>
LiveDataのLifecycleBoundObserverっていうインナークラスがLifecycleObserverを実装してる<br>
メインスレッドから呼び出さないと怒られるみたいね...<br>

~~~java
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            // 同じObserverは登録できないみたい
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
    // 中略
    class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
        @NonNull
        final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }

private abstract class ObserverWrapper {
    final Observer<? super T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        if (wasInactive && mActive) {
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            onInactive();
        }
        if (mActive) {
            dispatchingValue(this);
        }
    }
    }
~~~

