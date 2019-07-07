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

## LiveData#setValue

~~~java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

@SuppressWarnings("WeakerAccess") /* synthetic access */
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        // LifecycleBoundObserver(return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);)
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        // mLastVersionは初期値-1、setValueの際にincrement
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
~~~

## LiveData#postValue
別スレッドからメインスレッドにsetValueをメインスレッドにpost
~~~java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};
~~~

