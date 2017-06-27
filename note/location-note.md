# Android Location Note

## Note 1

使用 Google Play Service 提供的 Fused Location API，替代 Android 系统提供的原生 API。

学习文档：

1. [Building Apps with Location & Maps](https://developer.android.com/training/building-location.html)
1. [Reduce friction with the new Location APIs](https://android-developers.googleblog.com/2017/06/reduce-friction-with-new-location-apis.html)

### Getting the Last Known Location

#### Setup Google Play Services

首先要配置 Google Play Services，要先通过 SDK Manager 把 Google Repository 下载到要地，然后将其依赖添加到工程 module 级的 build.gradle 中。

    dependencies {
        compile 'com.google.android.gms:play-services-location:11.0.1'
    }

但是，要使用 Google Play Services，还有一个前提，手机必须是 4.0 以上 (这个能满足)，手机上必须安装了最新版的 Google Play Store App (这个就 shit 了，中国的手机基本不满足...)，因为 Google Play Services 提供的 API 实际上都是由 Google Play Store App 来完成实际工作的 (Why? 为什么要这么设计，shit again!)。

在使用这个 API 之前，都要先判断手机上的 Google Play Store App 有没有安装，能不能用，需不需要升级 (奇芭!)。有两种方法：

1. 使用 GoogleApiClient 类，注册 OnConnectionFailedListener，在 onConnectionFailed() 回调中处理 Google Play Services 无法使用的问题，包括三种结果：`SERVICE_MISSIONG`，`SERVICE_VERSION_UPDATE_REQUIRED`，`SERVICE_DISABLED`。
1. 使用 GoogleApiAvailability.getInstance().isGooglePlayServiceAvaiable() 方法判断，如果结果为 SUCCESS 则 Google Play Services API 可用，否则，可能得到上面所说的三种结果，用 getErrorDialog(errResult) 方法获取一个 Dialog 并显示给用户，提示用户安装或升级 Google Play Store App。

更详细的说明：[Accessing Google APIs](https://developers.google.com/android/guides/api-client)

![](https://developers.google.com/android/images/GoogleApiClient_2x.png)

#### 指定 Location Permission

Android 提供 2 种 location permission：`ACCESS_FINE_LOCATION` 和 `ACCESS_COARSE_LOCATION`，此例中我们只请求 `ACCESS_COARSE_LOCATION` 权限，这个权限只能得到精度比较低的 location。

从 Android 6.0 开始，部分权限还需要在程序中动态请求。

#### Create Location Services Client

在 Google Play Service 11.0.0 之前，正如上面所言，使用这些 API 之前，先要创建 GoogleApiClient，处理连接问题。但从 Google Play Service 11.0.0 开始，使用 Location API 时，在 [Reduce friction with the new Location APIs](https://android-developers.googleblog.com/2017/06/reduce-friction-with-new-location-apis.html) 这篇文章里说，不再需要 GoogleApiClient 类。直接使用 FusedLocationClient 就行。

示例代码：

    private FusedLocationProviderClient mFusedLocationClient;
    // ..
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // ...
        mFusedLocationClient = LocationServices.getFusedLocationProviderClient(this);
    }

#### Get the Last Know Location

使用 getLocation().addOnSuccessListener() 方法。

    mFusedLocationClient.getLastLocation()
            .addOnSuccessListener(this, new OnSuccessListener<Location>() {
                @Override
                public void onSuccess(Location location) {
                    // Got last known location. In some rare situations this can be null.
                    if (location != null) {
                        // ...
                    }
                }
            });

运行结果：

- 在我的已安装最新版 Google Play 的手机上，得到了 null 值的 location ...
- 在一台安装了老版本 Google Play 的手机上运行时，自动弹出了 Dialog 说道：您必须更新 Google Play 服务，然后才能运行 GoogleLocationSample。
