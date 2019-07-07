# LifeCycleまとめ
まとめたい

## lifecycle.addObserver(mMapsViewModel)

### lifecycleとは？
Kotlinで書いてるのでわかりにくいけど、FragmentActivity#getLifecycle()を呼び出している<br>
つまり、LifeCycle(LifeCycleRegistry)を取得している
~~~java
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @Override
    public Lifecycle getLifecycle() {
        return super.getLifecycle();
    }
~~~

~~~java
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
~~~

#### LifecycleRegistryとは？
Lifecycleを継承している<br>

## lifecycle.addObserver(mMapsViewModel)
※ViewModelにLifecycleObserverを実装している<br>
おそらくライフサイクルのコールバックが呼ばれるように設定している<br>
~~~java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
    // 中略
    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            // おそらくここでLifecycleObserverのライフサイクルに対応するメソッド呼び出している
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
~~~

~~~kotlin
    /**
     * LocationListener開始
     */
    @SuppressLint("MissingPermission")
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun addLocationListener() {
        mLocationManager = mContext.getSystemService(Context.LOCATION_SERVICE) as LocationManager
        mLocationManager?.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0F, mLocationListener)
        val lastLocation = mLocationManager?.getLastKnownLocation(LocationManager.GPS_PROVIDER)
        this.mLocationListener.onLocationChanged(lastLocation)
    }

    /**
     * LocationListener停止
     */
    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun removeLocationListener() {
        mLocationManager?.removeUpdates(this.mLocationListener)
        mLocationManager = null
    }
~~~