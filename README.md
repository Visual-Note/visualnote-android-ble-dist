# VisualnoteAndroidBle

## Requirements
<uses-permission android:name="android.permission.BLUETOOTH"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>

## Installation

include VisualnoteAndroidBleSdk-release.aar in your Android project

## API

First of all import the library
```java
import com.visualnote.visualnoteandroidble.sdk.BleManager;
import com.visualnote.visualnoteandroidble.sdk.Fretboard;
import com.visualnote.visualnoteandroidble.sdk.LedPoint;
import com.visualnote.visualnoteandroidble.sdk.VisualnoteBle;
```

then you must implement the listener, for example in a Fragment

```java
public interface VisualnoteBleListener {
    void onStartScan();
    void onConnect();
    void onDisconnect();
    void onSessionStarted();
    void onSessionTerminated();
    void onBLEStateChange(BleManager.BleState state);
    void onDeviceDiscovered(String device);
    void onScanTimeout();
}
```

ask the user permission on ACCESS_COARSE_LOCATION, and then init BLE

```java
public class FirstFragment extends Fragment {
    public void onViewCreated(@NonNull View view, Bundle savedInstanceState) {
        view.findViewById(R.id.button_init).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_init");
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                    requestPermissions(new String[]{Manifest.permission.ACCESS_COARSE_LOCATION}, PERMISSION_REQUEST_COARSE_LOCATION);
                }
            }
        });
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String permissions[], @NonNull int[] grantResults) {
        switch (requestCode) {
            case PERMISSION_REQUEST_COARSE_LOCATION: {
                getContext().getPackageManager();
                if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    Log.d("FirstFragment", "onRequestPermissionsResult PERMISSION_GRANTED");
                    VisualnoteBle.getInstance().initBle(getContext(), "MY-TOKEN");
                } else {
                    Log.d("FirstFragment", "onRequestPermissionsResult NOT GRANTED");
                }
            }
        }
    }    
}
```

to connect and send data, you must in sequence call:
```java
VisualnoteBle.getInstance().scan();                            // scan of available devices in the surrounding area
VisualnoteBle.getInstance().connect(deviceName, getContext()); // connect to the chosen device
```

those calls will have a callback each
```java
@Override
public View onCreateView(
        LayoutInflater inflater, ViewGroup container,
        Bundle savedInstanceState
) {
    VisualnoteBle.getInstance().setVisualnoteBleListener(new VisualnoteBle.VisualnoteBleListener() {
        @Override
        public void onStartScan() {
            Log.d(TAG, "::onStartScan");
        }

        @Override
        public void onConnect() {
            Log.d(TAG, "::onConnect");
        }

        @Override
        public void onDisconnect() {
            Log.d(TAG, "::onDisconnect");
        }

        @Override
        public void onSessionStarted() {
            Log.d(TAG, "::onSessionStarted");
        }

        @Override
        public void onSessionTerminated() {
            Log.d(TAG, "::onSessionTerminated");
        }

        @Override
        public void onBLEStateChange(BleManager.BleState state) {
            Log.d(TAG, "::onBLEStateChange " + state);
            switch (state) {
                case BLUETOOTH_ENABLED: {
                    Log.d("FirstFragment", "BLUETOOTH_ENABLED");
                    return;
                }
                case BLUETOOTH_NOT_ENABLED: {
                    Log.d("FirstFragment", "BLUETOOTH_NOT_ENABLED");
                    return;
                }
                case NO_BLE_DEVICE: {
                    Log.d("FirstFragment", "NO_BLE_DEVICE");
                    return;
                }
                case NO_BLUETOOTH_ADAPTER: {
                    Log.d("FirstFragment", "NO_BLUETOOTH_ADAPTER");
                }
                case DEVICE_NOT_FOUND: {
                    Log.d("FirstFragment", "DEVICE_NOT_FOUND");
                }
            }
        }

        @Override
        public void onDeviceDiscovered(String device) {
            Log.d(TAG, "::onDeviceDiscovered " + device);
            deviceName = device;
        }

        @Override
        public void onScanTimeout() {
            Log.d(TAG, "::onScanTimeout");
        }
    });
}
```

after connection you can start a session and send data
```java
VisualnoteBle.getInstance().startSession();

VisualnoteBle.getInstance().sendSingleLed(new LedPoint(20, 1, Fretboard.Finger.index));
VisualnoteBle.getInstance().sendManyLeds(new LedPoint[]{new LedPoint(1, 1, Fretboard.Finger.index), new LedPoint(2, 2, Fretboard.Finger.index)});

// these are the available
// finger:colors
public enum Finger {
    thumb(0),
    index(1),
    middle(2),
    ring(3),
    pinkie(4),
    open(5),
    defaultColor(6);
}
```

when you're done playing a song you can close the session, so to minimize battery usage
```java
VisualnoteBle.getInstance().stopSession();
```

## Full Example
```java
import android.Manifest;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Bundle;
import android.os.Handler;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;

import androidx.annotation.NonNull;
import androidx.fragment.app.Fragment;
import androidx.navigation.fragment.NavHostFragment;

import com.visualnote.visualnoteandroidble.sdk.BleManager;
import com.visualnote.visualnoteandroidble.sdk.Fretboard;
import com.visualnote.visualnoteandroidble.sdk.LedPoint;
import com.visualnote.visualnoteandroidble.sdk.VisualnoteBle;

public class FirstFragment extends Fragment {
    private static final String TAG = "FirstFragment";
    private static final int PERMISSION_REQUEST_COARSE_LOCATION = 456;
    private String deviceName;

    Handler timer = new Handler();

    private final Runnable runTask = new Runnable() {
        @Override
        public void run() {
            Log.d(TAG, "::timer sendNotes");
            sendNotes();
            timer.postDelayed(this, 250);
        }
    };

    private void sendNotes() {
        Log.d(TAG, "::sendNotes");
        for(int i = 0; i < 24; i++){
            VisualnoteBle.getInstance().sendSingleLed(new LedPoint(i, 1, Fretboard.Finger.index));
        }
    }

    @Override
    public View onCreateView(
            LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState
    ) {
        VisualnoteBle.getInstance().setVisualnoteBleListener(new VisualnoteBle.VisualnoteBleListener() {
            @Override
            public void onStartScan() {
                Log.d(TAG, "::onStartScan");
            }

            @Override
            public void onConnect() {
                Log.d(TAG, "::onConnect");
            }

            @Override
            public void onDisconnect() {
                Log.d(TAG, "::onDisconnect");
            }

            @Override
            public void onSessionStarted() {
                Log.d(TAG, "::onSessionStarted");
            }

            @Override
            public void onSessionTerminated() {
                Log.d(TAG, "::onSessionTerminated");
            }

            @Override
            public void onBLEStateChange(BleManager.BleState state) {
                Log.d(TAG, "::onBLEStateChange " + state);
                switch (state) {
                    case BLUETOOTH_ENABLED: {
                        Log.d("FirstFragment", "BLUETOOTH_ENABLED");
                        return;
                    }
                    case BLUETOOTH_NOT_ENABLED: {
                        Log.d("FirstFragment", "BLUETOOTH_NOT_ENABLED");
                        return;
                    }
                    case NO_BLE_DEVICE: {
                        Log.d("FirstFragment", "NO_BLE_DEVICE");
                        return;
                    }
                    case NO_BLUETOOTH_ADAPTER: {
                        Log.d("FirstFragment", "NO_BLUETOOTH_ADAPTER");
                    }
                    case DEVICE_NOT_FOUND: {
                        Log.d("FirstFragment", "DEVICE_NOT_FOUND");
                    }
                }
            }

            @Override
            public void onDeviceDiscovered(String device) {
                Log.d(TAG, "::onDeviceDiscovered " + device);
                deviceName = device;
            }

            @Override
            public void onScanTimeout() {
                Log.d(TAG, "::onScanTimeout");
            }
        });

        // Inflate the layout for this fragment
        return inflater.inflate(R.layout.fragment_first, container, false);
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String permissions[], @NonNull int[] grantResults) {
        switch (requestCode) {
            case PERMISSION_REQUEST_COARSE_LOCATION: {
                getContext().getPackageManager();
                if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    Log.d("FirstFragment", "onRequestPermissionsResult PERMISSION_GRANTED");
                    VisualnoteBle.getInstance().initBle(getContext(), "MY-TOKEN");
                } else {
                    Log.d("FirstFragment", "onRequestPermissionsResult NOT GRANTED");
                }
            }
        }
    }

    public void onViewCreated(@NonNull View view, Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        view.findViewById(R.id.button_init).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_init");
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                    requestPermissions(new String[]{Manifest.permission.ACCESS_COARSE_LOCATION}, PERMISSION_REQUEST_COARSE_LOCATION);
                }
            }
        });

        view.findViewById(R.id.button_scan).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_scan");
                VisualnoteBle.getInstance().scan();
            }
        });

        view.findViewById(R.id.button_connect).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_connect");
                VisualnoteBle.getInstance().connect(deviceName, getContext());
            }
        });

        view.findViewById(R.id.button_disconnect).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_disconnect");
                VisualnoteBle.getInstance().disconnect();
            }
        });

        view.findViewById(R.id.button_start_session).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_start_session");
                VisualnoteBle.getInstance().startSession();
            }
        });

        view.findViewById(R.id.button_send).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_send");
                // VisualnoteBle.getInstance().sendSingleLed(new LedPoint(20, 1, Fretboard.Finger.index));
                timer.post(runTask);
            }
        });

        view.findViewById(R.id.button_stop_session).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("FirstFragment:onClick", "button_stop_session");
                timer.removeCallbacks(runTask);
                VisualnoteBle.getInstance().stopSession();
            }
        });
    }
}
```

## Author

Visual Note SRL, giona.granata@gmail.com

## License

Copyright (c) 2021 Visual Note SRL

Authorized Users only.
Fully protected copyright VisualNote SRL
