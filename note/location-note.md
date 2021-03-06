# Android Location Note

实现获取定位的几种方法：

1. Android 原生 Location API
1. Google Play LocationService API (国内不可用)
1. 百度或高德的地图 SDK (国内推荐使用)

## Note 1

Android 原生 Location API。

学习文档：

1. [Location and Sensors APIs](https://developer.android.com/guide/topics/sensors/index.html)
1. [Location and Maps](https://developer.android.com/guide/topics/location/index.html)
1. [Location Strategies](https://developer.android.com/guide/topics/location/strategies.html)
1. [Android 地理位置服务解析](http://unclechen.github.io/2016/09/02/Android%E5%9C%B0%E7%90%86%E4%BD%8D%E7%BD%AE%E6%9C%8D%E5%8A%A1%E8%A7%A3%E6%9E%90/) (下面许多内容来自此文)

三种定位方式：GPS，WIFI，基站。

三种 PROVIDER：

1. `LocationManager.GPS_PROVIDER` - 通过 GPS 定位，精度高，但耗电和耗时。
1. `LocationManager.NETWORK_PROVIDER` - 通过 WIFI 或基站定位，获取速度快，但精度比 GPS 低。
1. `LocationManager.PASSIVE_PROVIDER` - 被动的接收更新的地理位置信息，而不用自己主动请求地理位置。意思就是共享手机上其他App采集的位置信息，而不是自己主动去采集。

选择 provider 的策略：

1. 如果应用只是偶尔用一下定位，不考虑省电的情况，可以直接指定使用某个 provider，简单粗暴，反正总共就 3 个，常用的就 2 个，不需要太高精确度就选 `NETWORK_PROVIDER`，考虑精确度就选 `GPS_PROVIDER`。
1. 稍微智能一点，根据当前用户设置来选择某个 provider，比如用户如果 GPS 没开那就选 `NETWORK_PROVIDER` 好了，或者提示用户把开关打开；如果 `GPS_PROVIDER` 某个时长内没有获取到 location，就再尝试用`NETWORK_PROVIDER`。
1. 如果再要考虑到省电的情况 (比如要长时间监听 location 变化)，那么就要根据更多的情况来选择某个 provider，比如电量，用户设置等情况。监听用户手机的系统状况，比如当电量低于 10% 时，切换成 `PASSIVE_PROVIDER`；通过 sensor 监听手机的活动情况，如果长时间处于 still 状态，就停止监听或降低监听的频率，如果处于 driving 的状态，就提高监听的频率和减小监听的范围。
1. 使用 Criteria 类，设置希望的要求，让系统帮我们选一个 provider。

        Criteria criteria = new Criteria();
        criteria.setAccuracy(Criteria.ACCURACY_FINE);           // 设置定位精准度
        criteria.setAltitudeRequired(false);                    // 是否要求海拔
        criteria.setBearingRequired(true);                      // 是否要求方向
        criteria.setCostAllowed(true);                          // 是否要求收费
        criteria.setSpeedRequired(true);                        // 是否要求速度
        criteria.setPowerRequirement(Criteria.POWER_LOW);       // 设置相对省电
        criteria.setBearingAccuracy(Criteria.ACCURACY_HIGH);    // 设置方向精确度
        criteria.setSpeedAccuracy(Criteria.ACCURACY_HIGH);      // 设置速度精确度
        criteria.setHorizontalAccuracy(Criteria.ACCURACY_HIGH); // 设置水平方向精确度
        criteria.setVerticalAccuracy(Criteria.ACCURACY_HIGH);   // 设置垂直方向精确度

        // 返回满足条件的，当前设备可用的 location provider
        // 当第 2 个参数为 false 时，返回当前设备所有 provider 中最符合条件的那个（但是不一定可用）
        // 当第 2 个参数为 true 时，返回当前设备所有可用的 provider 中最符合条件的那个
        String rovider  = mLocationManager.getBestProvider(criteria, true);

### 基本使用

获得 LocationManager 实例，创建 LocationListener，选择合适的 provider，调用 locationManager.requestLocationUpdate() 来监听 location 变化。

    // 获得 LocationManager 的实例
    LocationManager locationManager = (LocationManager)this.getSystemService(Context.LOCATION_SERVICE);

    // 定义一个监听器，实现 onLocationChanged 方法，在这个方法里面可以拿到更新后的地理位置
    LocationListener locationListener = new LocationListener() {
        public void onLocationChanged(Location location) {
            // 新的 Location 值在这里返回，Location 实例中包含着纬度、经度、海拔、精确度、更新时间等一系列信息。
            makeUseOfNewLocation(location);
        }
        public void onStatusChanged(String provider, int status, Bundle extras) {}
        public void onProviderEnabled(String provider) {}
        public void onProviderDisabled(String provider) {}
    };

    // 注册监听器，当地理位置变化时，发出通知给 Listener。这个方法很关键。4 个参数需要了解清楚：
    // 第 1 个参数：你所使用的 provider 名称，是个 String
    // 第 2 个参数 minTime：地理位置更新时发出通知的最小时间间隔
    // 第 3 个参数 minDistance：地理位置更新发出通知的最小距离，第 2 和第 3 个参数的作用关系是“或”的关系，也就是满足任意一个条件都会发出通知。这里第 2、3 个参数都是 0，意味着任何时间，只要位置有变化就会发出通知。
    // 第 4 个参数：你的监听器
    locationManager.requestLocationUpdates(LocationManager.NETWORK_PROVIDER, 0, 0, locationListener);

不需要的时候及时地调用 locationManager.removeUpdate(listener) 停止监听。

通过 locationManager.getLastKnownLocation(provider) 取得系统上次缓存的地理位置。(虽然是缓存的值，但也是需要 permission 的)。

### 优化策略

上面关于 provider 的选择其实已有所涉及，再看 [Android 地理位置服务解析](http://unclechen.github.io/2016/09/02/Android%E5%9C%B0%E7%90%86%E4%BD%8D%E7%BD%AE%E6%9C%8D%E5%8A%A1%E8%A7%A3%E6%9E%90/) 这篇文章，写得蛮详细的，包括官方文档的翻译内容。

## Note 2

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

- 在我的已安装最新版 Google Play 的 Meizu MX4 Pro (Android 5.1.1) 上，得到了 null 值的 location，且没有弹出需要获取位置权限的通知 (对国产手机的兼容性极差)
- 在一台安装了老版本 Google Play 的 Samsung 手机上运行时，自动弹出了 Dialog 说道：您必须更新 Google Play 服务，然后才能运行 GoogleLocationSample。
- 在安装了最新版的 Google Play 的 OnePlus 3T (android 7.1) 上运行，一切正常。

### Changing Location Settings & Receiving Location Updates

**初步测试发现，此功能需要 VPN 支持**

监听 location 的流程：

1. 动态检查并请求 location 权限

1. 检查当前手机的设置是否能够符合我们想要请求的 location 的精度要求。比如通过 蜂窝/WIFI/GPS 都可以取到 location，但前二者的精度较低，GPS 的精度较高。如果我们想请求高精度的 location，却发现只有 wifi 和蜂窝的开关打开了，GPS 的开关没有打关，这样的设置就不符合我们的要求，此时，我们就需要请求用户打开 GPS 开关。

   这里面涉及到三个类，LocationRequest，表示我们对 location 的要求，主要是精度要求，有四种值，后面再讲。LocationSettingRequest，这个请求用来查询当前手机设置与我们想要的 LocationRequest 是否匹配，所以，这个 LocationSettingRequest 自然是需要和 LocationRequest 关联的，因此 LocationSettingRequest 是这样生成的：

        mLocationRequest = new LocationRequest();
        mLocationRequest.setInterval(UPDATE_INTERVAL_IN_MILLISECONDS);
        mLocationRequest.setFastestInterval(FASTEST_UPDATE_INTERVAL_IN_MILLISECONDS);
        mLocationRequest.setPriority(LocationRequest.PRIORITY_HIGH_ACCURACY);

        mLocationSettingsRequest = new LocationSettingsRequest.Builder()
                .addLocationRequest(mLocationRequest)
                .build();

   然后，我们用 SettingClient 来实现设置的查询：

        SettingsClient client = LocationServices.getSettingsClient(this);
        Task<LocationSettingsResponse> task = client.checkLocationSettings(mLocationSettingsRequest);

   SettingsClient.checkLocationSettings() 返回值是一个 task，我们给 task 注册成功和失败的监听器，如果产生失败的回调，说明当前手机设置没有达到我们的 location 要求，需要更改手机设置，因此我们要弹出 dialog 提示用户去修改设置，打开更多开关。如果产生成功的回调，说明当前手机设置是 OK 的，不需要修改。

1. 在上一步 task 的成功回调中，用 FusedLocationProviderClient.requestLocationUpdates() 去请求并监听 location 的变化，其中第一个参数 LocationRequest 必须和上一步中查询设置所用到的 LocationRequest 保持一致。

        mFusedLocationClient = LocationServices.getFusedLocationProviderClient(this);
        mLocationCallback = new LocationCallback() {
            @Override
            public void onLocationResult(LocationResult locationResult) {
                super.onLocationResult(locationResult);
                mCurrentLocation = locationResult.getLastLocation();
                // ...
                // updateUI();
            }
        };

        task.addOnSuccessListener(this, new OnSuccessListener<LocationSettingsResponse>() {
                    @SuppressWarnings("MissingPermission")
                    @Override
                    public void onSuccess(LocationSettingsResponse locationSettingsResponse) {
                        mFusedLocationClient.requestLocationUpdates(mLocationRequest,
                                mLocationCallback, Looper.myLooper());
                        //...
                    }
                })
            .addOnFailureListener(...)

整个流程还是有点麻烦的，麻烦也还好啦，但最关键的是，测试后发现，这套 API 在我的国产手机 Meizu Android 5.0 上基本不 work (尽管已经安装了最新版的 Google Play)，也没有什么错误，但就是得不到结果...在 OnePlus 3T Android 7.0 上是正常的。

LocationRequest 的精度要求的四个级别：

- `PRIORITY_BALANCED_POWER_ACCURACY`: 大概是一个街区 100 米的精度，可能使用 WIFI 或 cell，coarse 的精度
- `PRIORITY_HIGH_ACCURACY`: 可能使用 GPS，一步的精度?
- `PRIORITY_LOW_POWER`: 10 千米的精度
- `PRIORITY_NO_POWER`: 不会主动获取 location，只会被动的接受别的 app 获取的 location

googlesamples 里还有两个例子：

- Location Updates using a PendingIntent: Get updates about a device's location using a PendingIntent. Sample shows implementation using an IntentService as well as a BroadcastReceiver. 在 FusedLocationProviderClient.requestLocationUpdates(locationRequest, pendingIntent) 方法中，没有使用 callback 来接收回调，而是用 pendingIntent 来接收新的 location。
- Location Updates using a Foreground Service: Get updates about a device's location using a bound and started foreground service. 在 Foreground Service 中监听 location。

### Displaying a Location Address

**初步测试发现，此功能需要 VPN 支持，holy shit! Geocoder 不是 android 框架的内容吗? 这也不能用，太夸张了吧**

将地址转换成 location 坐标，这个过程叫 geocoding，将 location 坐标转换成地址，这个过程叫 reverse geocoding，Android 框架中提供了原生 API Geocoder 来处理这个事情，使用 Geocoder.getFromLocation() 方法。

Geocode 是同步调用，而它对 location 进行转换又是一个耗时操作，所以要把它放到一个单独的线程里工作，可以选择 AsyncTask 或 IntentService，但前者和 Activity 的生命周期相关，如果 Activity 中途被重建了，AsyncTask 的结果就不会更新到新的 Activity 上，所以文档中选择了使用 IntentService 来在后台进行转换，通过 ResultReceiver 回传 address 结果。(注意，这里的 ResultReceiver 只是一个普通的 Parcelable，跟 BroadcastReceiver 没有关系)。

核心代码：

    try {
        addresses = geocoder.getFromLocation(
                location.getLatitude(),
                location.getLongitude(),
                // In this sample, get just a single address.
                1);
    } catch (IOException ioException) {
        // Catch network or other I/O problems.
        errorMessage = getString(R.string.service_not_available);
        Log.e(TAG, errorMessage, ioException);
    } catch (IllegalArgumentException illegalArgumentException) {
        // Catch invalid latitude or longitude values.
        errorMessage = getString(R.string.invalid_lat_long_used);
        Log.e(TAG, errorMessage + ". " +
                "Latitude = " + location.getLatitude() +
                ", Longitude = " +
                location.getLongitude(), illegalArgumentException);
    }

### Creating and Monitoring Geofences

The latitude, longitude, and radius define a geofence, creating a circular area, or fence, around the location of interest.

You can have multiple active geofences, with a limit of 100 per device user. For each geofence, you can ask Location Services to send you entrance and exit events, or you can specify a duration within the geofence area to wait, or dwell, before triggering an event. You can limit the duration of any geofence by specifying an expiration duration in milliseconds. After the geofence expires, Location Services automatically removes it.

![](https://developer.android.com/images/training/geofence.png)

我的理解：Geofence 就是你划定的一块范围，然后你可以监听当前 location 的变化，如果当前 location 进入或离开，或停留在刚才划定的那块范围内，就会收到相应的事件通知。

运行本小节的[示例工程](https://github.com/googlesamples/android-play-location/tree/master/Geofencing)，因为程序中默认划定的区域是硅谷，所以你除非在硅谷，否则测试时永远收不到事件通知，修改代码，加上当前你所在位置的区域，就能收到通知了。

    // Constants.java
    static {
        // San Francisco International Airport.
        BAY_AREA_LANDMARKS.put("SFO", new LatLng(37.621313, -122.378955));

        // Googleplex.
        BAY_AREA_LANDMARKS.put("GOOGLE", new LatLng(37.422611,-122.0840577));

        // MyLocation
        BAY_AREA_LANDMARKS.put("BAOSHAN", new LatLng(31.323331,121.392625));
    }

收到的通知：

![](./art/geofence_notification.png)

#### Set up for Geofence Monitoring

1. 在 AndroidManifest.xml 中声明 `android.permission.ACCESS_FINE_LOCATION` 权限。

1. 在 AndroidManifest.xml 中添加用于处理 Geofence 事件的 IntentService。

        <application
            android:allowBackup="true">
            ...
            <service android:name=".GeofenceTransitionsIntentService"/>
        <application/>

1. 创建 GeofencingClient 实例。

        private GeofencingClient mGeofencingClient;
        // ...
        mGeofencingClient = LocationServices.getGeofencingClient(this);

#### Create and Add Geofences

创建 Geofence，Geofence 就是一块区域范围，是圆形的，指定经纬度作为圆心，再指定一个半径作为范围，还可以指定监听这个 Geofence 的变化类型，是监听进入，还是离开，还是停留，或者是都监听。Geofence 可以由 Geofence.Builder 来生成。

可以创建多个 Geofence，把它们加入一个列表中，然后用这个列表去创建 GeofenceRequest，GeofenceRequest 可以用 GeofencingRequest.Builder 来生成。

之后再创建 PendingIntent 来处理 Geofence 的变化事件。

最后，把 GeofenceRequest 和 PendingIntent 加入到 GeofencingClient 中，开始监听 Geofence 事件。

##### Create geofence objects

    mGeofenceList.add(new Geofence.Builder()
        // Set the request ID of the geofence. This is a string to identify this
        // geofence.
        .setRequestId(entry.getKey())

        .setCircularRegion(
                entry.getValue().latitude,
                entry.getValue().longitude,
                Constants.GEOFENCE_RADIUS_IN_METERS
        )
        .setExpirationDuration(Constants.GEOFENCE_EXPIRATION_IN_MILLISECONDS)
        .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_ENTER |
                Geofence.GEOFENCE_TRANSITION_EXIT)
        .build());

在这个例子中，Geofence 的经纬度是个常量，实际项目中，这个值会根据我们的当前位置来动态生成。

##### Specify geofences and initial triggers

    private GeofencingRequest getGeofencingRequest() {
        GeofencingRequest.Builder builder = new GeofencingRequest.Builder();
        builder.setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER);
        builder.addGeofences(mGeofenceList);
        return builder.build();
    }

`INITIAL_TRIGGER_ENTER` 表示，当监听开始时，如果当前位置已处于所划定的 Geofence 中时，会触发 Geofence 的 `GEOFENCE_TRANSITION_ENTER` 事件。

为是减少 spam 打扰及减少电池消耗，更推存使用 `INITIAL_TRIGGER_DWELL`，它表示，只有当你在某个 Geofence 中停留超过指定的时长后，才会触发事件。

另外，当 Geofence 的范围设置为不小于 100 米，也可以减少电池消耗，及减小 WIFI 的 location 精确度的影响。

##### Define an Intent for geofence transitions

    public class MainActivity extends AppCompatActivity {
        // ...
        private PendingIntent getGeofencePendingIntent() {
            // Reuse the PendingIntent if we already have it.
            if (mGeofencePendingIntent != null) {
                return mGeofencePendingIntent;
            }
            Intent intent = new Intent(this, GeofenceTransitionsIntentService.class);
            // We use FLAG_UPDATE_CURRENT so that we get the same pending intent back when
            // calling addGeofences() and removeGeofences().
            mGeofencePendingIntent = PendingIntent.getService(this, 0, intent, PendingIntent.
                    FLAG_UPDATE_CURRENT);
            return mGeofencePendingIntent;
    }

##### Add geofences

    mGeofencingClient.addGeofences(getGeofencingRequest(), getGeofencePendingIntent())
        .addOnSuccessListener(this, new OnSuccessListener<Void>() {
            @Override
            public void onSuccess(Void aVoid) {
                // Geofences added
                // ...
            }
        })
        .addOnFailureListener(this, new OnFailureListener() {
            @Override
            public void onFailure(@NonNull Exception e) {
                // Failed to add geofences
                // ...
            }
        });

#### Handle Geofence Transitions

When Location Services detects that the user has entered or exited a geofence, it sends out the Intent contained in the PendingIntent you included in the request to add geofences. This Intent is received by a service like GeofenceTransitionsIntentService, which obtains the geofencing event from the intent, determines the type of Geofence transition(s), and determines which of the defined geofences was triggered. It then sends a notification as the output.

当 Location Service 检测到当前用户进入或离开一个 Geofence 时，它会发出一个注册在 GeofencingClient 中的 PendingIntent，这个 Intent 将会交给指定的 IntentService 处理，这个 intent 中携带了触发 Geofence 的事件类型等数据。

    public class GeofenceTransitionsIntentService extends IntentService {
        // ...
        protected void onHandleIntent(Intent intent) {
            GeofencingEvent geofencingEvent = GeofencingEvent.fromIntent(intent);
            if (geofencingEvent.hasError()) {
                String errorMessage = GeofenceErrorMessages.getErrorString(this,
                        geofencingEvent.getErrorCode());
                Log.e(TAG, errorMessage);
                return;
            }

            // Get the transition type.
            int geofenceTransition = geofencingEvent.getGeofenceTransition();

            // Test that the reported transition was of interest.
            if (geofenceTransition == Geofence.GEOFENCE_TRANSITION_ENTER ||
                    geofenceTransition == Geofence.GEOFENCE_TRANSITION_EXIT) {

                // Get the geofences that were triggered. A single event can trigger
                // multiple geofences.
                List<Geofence> triggeringGeofences = geofencingEvent.getTriggeringGeofences();

                // Get the transition details as a String.
                String geofenceTransitionDetails = getGeofenceTransitionDetails(
                        this,
                        geofenceTransition,
                        triggeringGeofences
                );

                // Send notification and log the transition details.
                sendNotification(geofenceTransitionDetails);
                Log.i(TAG, geofenceTransitionDetails);
            } else {
                // Log the error.
                Log.e(TAG, getString(R.string.geofence_transition_invalid_type,
                        geofenceTransition));
            }
        }

#### Stop Geofence Monitoring

调用 GeofencingClient 实例的 removeGeofences() 方法。有两种实现，一种是参数是 PendingIntent，一种是参数是 List<String>，表示 Geofences 的 RequestId。

#### Use Best Practices for Geofencing

略，用到时再回来细看。总体而言，主要是选择最优的半径，Geofence 的半径不要太小，尽量选择 DWELL 的事件类型，替代 ENTER。

(貌似 Geofence 的功能主要是依赖 WIFI 来定位，而不是 GPS??，因为 WIFI 的精度已经能达到 100m，而 Geofence 推荐的最小半径就是 100m)。

### Recognizing the User's Current Activity

这部分内容只有[示例代码](https://github.com/googlesamples/android-play-location/tree/master/ActivityRecognition)，没有详细的使用介绍。

示例代码运行结果：

![](./art/activity_recognition.png)

[ActivityRecognitionApi](https://developers.google.com/android/reference/com/google/android/gms/location/ActivityRecognitionApi) 可以识别用户当前进行的活动，比如是在走路啊，还是开车，或是静止。

看了文档和代码，用起来比较简单，API 也很少，和 Geofence 的用法很相似，首先向 GoogleApiClient 注册监听 Activity 的变化，然后 Activity 的变化会通过 Intent 发送到 IntentService 处理。IntentService 在 handleIntent() 中可以将结果发送到通知栏，或是通过广播发送给 BroadcastReceiver。

ActivityRecognitionApi 需要 `com.google.android.gms.permission.ACTIVITY_RECOGNITION` 权限，只需在 AndroidManifest.xml 中声明一下就行，不用动态申请，我猜想，亦从文档中得知，这个活动检测是通过传感器来实现的，跟 location 没有关系，因为此示例代码并没有申请 location 权限。

> he activities are detected by periodically waking up the device and reading short bursts of sensor data. It only makes use of low power sensors in order to keep the power usage to a minimum.

#### ActivityRecognitionApi 的使用

**连接 GoogleApiClient** 

(我猜以后应该也可以直接用 LocationService 来替代)

    protected synchronized void buildGoogleApiClient() {
        mGoogleApiClient = new GoogleApiClient.Builder(this)
                .enableAutoManage(this, this)
                .addConnectionCallbacks(this)
                .addOnConnectionFailedListener(this)
                .addApi(ActivityRecognition.API)
                .build();
    }

**定义用来处理结果的 PendingIntent 和 IntentService**

    /**
     * Gets a PendingIntent to be sent for each activity detection.
     */
    private PendingIntent getActivityDetectionPendingIntent() {
        Intent intent = new Intent(this, DetectedActivitiesIntentService.class);

        // We use FLAG_UPDATE_CURRENT so that we get the same pending intent back when calling
        // requestActivityUpdates() and removeActivityUpdates().
        return PendingIntent.getService(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
    }

    public class DetectedActivitiesIntentService extends IntentService {
        ...
    }

**请求监听 Activity 变化**

    public void requestActivityUpdatesButtonHandler(View view) {
        if (!mGoogleApiClient.isConnected()) {
            Toast.makeText(this, getString(R.string.not_connected),
                    Toast.LENGTH_SHORT).show();
            return;
        }
        ActivityRecognition.ActivityRecognitionApi.requestActivityUpdates(
                mGoogleApiClient,
                Constants.DETECTION_INTERVAL_IN_MILLISECONDS,
                getActivityDetectionPendingIntent()
        ).setResultCallback(this);
    }

`setResultCallback(this)` 的 callback 用来处理是否注册或者移除监听器成功。

**操作结果**

    // DetectedActivitiesIntentService.java
    protected void onHandleIntent(Intent intent) {
        ActivityRecognitionResult result = ActivityRecognitionResult.extractResult(intent);
        Intent localIntent = new Intent(Constants.BROADCAST_ACTION);

        // Get the list of the probable activities associated with the current state of the
        // device. Each activity is associated with a confidence level, which is an int between
        // 0 and 100.
        ArrayList<DetectedActivity> detectedActivities = (ArrayList) result.getProbableActivities();

        // Log each activity.
        Log.i(TAG, "activities detected");
        for (DetectedActivity da: detectedActivities) {
            Log.i(TAG, Constants.getActivityString(
                            getApplicationContext(),
                            da.getType()) + " " + da.getConfidence() + "%"
            );
        }

        // Broadcast the list of detected activities.
        localIntent.putExtra(Constants.ACTIVITY_EXTRA, detectedActivities);
        LocalBroadcastManager.getInstance(this).sendBroadcast(localIntent);
    }

（如果不能用 GooglePlay，那有没有第三方实现此功能的库啊??)

## Note 3

使用 Google Maps API。

学习文档：
1. [Adding Maps](https://developer.android.com/training/maps/index.html)
1. [Maps API Getting Started](https://developers.google.com/maps/documentation/android-api/start)

### 使用入门

创建 Android Studio 工程，选择 Google Maps Activity 模板，将会自动生成关于 Maps 的代码。

根据生成代码中的提示，从 Google Console 中创建 Maps API 的秘钥，然后再填回工程中。

That's all，然后工程就可以跑起来了。

核心代码：

    public class MapsActivity extends FragmentActivity implements OnMapReadyCallback {

        private GoogleMap mMap;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_maps);
            SupportMapFragment mapFragment = (SupportMapFragment) getSupportFragmentManager()
                    .findFragmentById(R.id.map);
            mapFragment.getMapAsync(this);
        }

        @Override
        public void onMapReady(GoogleMap googleMap) {
            mMap = googleMap;

            // Add a marker in Sydney, Australia, and move the camera.
            LatLng sydney = new LatLng(-34, 151);
            mMap.addMarker(new MarkerOptions().position(sydney).title("Marker in Sydney"));
            mMap.moveCamera(CameraUpdateFactory.newLatLng(sydney));
        }
    }

### 创建地图 - 地图对象

重要的内容，讲解了 GoogleMap 对象的一些配置 api，比如地图类型，摄像头初始位置...

1. 展示地图的容器：MapFragment 或 MapView，MapView 与 MapFragment 很相似，它也充当地图容器，用来展示地图，通过 GoogleMap 对象公开核心地图功能。(我觉得 MapFragment 里面就是包了一个 MapView 对象，它管理了 MapView 的生命周期)。
1. 地图对象 GoogleMap，由上例可知，可以 onMapReady(GoogleMap googleMap) 回调中取得，回调通过 MapFragment/MapView 的 getMapAsync() 方法注册。

### 在地图上绘制

#### 标记

详细介绍 `GoogleMap.addMarker(markerOptions)` 的具体使用。

其它略，需要时再回来细看。

#### 形状

Google Maps API for Android 提供了一些简单的方法，让您可以方便地向地图添加形状，以针对您的应用对地图进行定制。

- Polyline 是一系列相连的线段，可组成您想要的任何形状，并可用于在地图上标记路径和路线
- Polygon 是一种封闭形状，可用于在地图上标记区域
- Circle 是投影在绘制于地图上的地球表面上地理位置准确的圆圈

对于所有上述形状，您都可以通过改变若干属性来定制其外观。

### 与地图交互 - 位置数据

只有需要在地图上显示当前位置时，我们才需要 location permission，否则，单独使用地图是不需要地理位置权限的。

`mMap.setMyLocationEnabled(true)` 启用 My Location 层，My Location 按钮会出现在地图的右上角。 当用户点击该按钮时，摄像头将设备的当前位置（若已知）显示为地图的中心。 设备处于静止状态时，地图以小蓝点指示该位置；处于运动状态时则以 V 形指示该位置。

虽然点击此按钮后会定位到当前位置，但 map 并没有提供 api 去获取这个位置的具体值，你还是需要自己手动通过 Location API 去拿到当前位置的值。

(咦，GoogleMaps API 并不需要处理 GoogleApiClient 连接的问题... 嗯，也是可以理解的，这部分功能应该是 GoogleMaps SDK 独立处理的)

## Note 4

开源项目学习：

1. [mauron85/react-native-background-geolocation](https://github.com/mauron85/react-native-background-geolocation)
1. [mrmans0n/smart-location-lib](https://github.com/mrmans0n/smart-location-lib)
