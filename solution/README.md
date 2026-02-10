# How to Solve the Challenge?

In the first step we install the application on an android emulator. we can see that the application have no functionalities and there is no function available for adding notes:
<img width="1080" height="2400" alt="Screenshot_1770687734" src="https://github.com/user-attachments/assets/5a7aaaa3-43b9-4ac4-94c3-7c46f29b2946" />

so what is the next step? the next step is reverse engineering and understanding the flow of the application. we can reverse it using jadx-gui.
https://github.com/skylot/jadx

the first step of android reverse engineering is checking the AndroidManifest.xml file for gathering information about some core configurations and application components. so let's see what is inside this file:
<img width="1557" height="1371" alt="image" src="https://github.com/user-attachments/assets/537badb8-b22e-4cf3-a648-c8e1c3ae1e4a" />

as we can see, the MainActivity class is set as the entrypoint of the application and also we have another class called MemoProvider which is an exported component (In android, exported components are meant to be accessible outside of the application sandboxing) which we will discuss about it later.
let's analyze the MainActivity:
<img width="1907" height="1388" alt="image" src="https://github.com/user-attachments/assets/57c70833-cd9b-422d-810c-e44adfd65936" />

```Java
package com.athack.notebox;

import android.app.Activity;
import android.database.Cursor;
import android.os.Bundle;
import android.util.Log;
import android.webkit.JavascriptInterface;
import android.webkit.WebView;
import com.athack.notebox.vault.Vault;
import java.io.IOException;
import java.util.Arrays;
import java.util.UUID;
import p002b.InterfaceC0082a;
import p003b1.AbstractC0083a;
import p017i0.C0184c;

/* compiled from: r8-map-id-7531e992cccfa3e53390e2ae6cf6770ec1e3790d4e3ad2079e197779663f45f4 */
/* loaded from: classes.dex */
public final class MainActivity extends Activity {

    /* renamed from: a */
    public String f235a;

    /* compiled from: r8-map-id-7531e992cccfa3e53390e2ae6cf6770ec1e3790d4e3ad2079e197779663f45f4 */
    @InterfaceC0082a
    public final class AppInterface {
        public AppInterface() {
        }

        @JavascriptInterface
        public final String getAppVersion() {
            return "2.1.0-build44";
        }

        @JavascriptInterface
        public final void logAnalytics(String str) {
            str.getClass();
            Log.d("SYS", "Event: ".concat(str));
        }

        @JavascriptInterface
        public final String validateSession(String str) {
            str.getClass();
            String str2 = MainActivity.this.f235a;
            if (str2 != null) {
                if (!str2.equals(str)) {
                    return "Error: 401 Unauthorized Session";
                }
                MainActivity mainActivity = MainActivity.this;
                mainActivity.getClass();
                return Vault.INSTANCE.open(mainActivity);
            }
            C0184c c0184c = new C0184c("lateinit property appSessionId has not been initialized");
            String name = AbstractC0083a.class.getName();
            StackTraceElement[] stackTrace = c0184c.getStackTrace();
            int length = stackTrace.length;
            int i2 = -1;
            for (int i3 = 0; i3 < length; i3++) {
                if (name.equals(stackTrace[i3].getClassName())) {
                    i2 = i3;
                }
            }
            c0184c.setStackTrace((StackTraceElement[]) Arrays.copyOfRange(stackTrace, i2 + 1, length));
            throw c0184c;
        }
    }

    @Override // android.app.Activity
    public final void onCreate(Bundle bundle) throws IOException {
        super.onCreate(bundle);
        setContentView(R.layout.activity_main);
        WebView webView = (WebView) findViewById(R.id.webView);
        String string = UUID.randomUUID().toString();
        string.getClass();
        this.f235a = string;
        webView.getSettings().setJavaScriptEnabled(true);
        WebView.setWebContentsDebuggingEnabled(false);
        webView.setWebChromeClient(new C0095a());
        webView.addJavascriptInterface(new AppInterface(), "AndroidSys");
        StringBuilder sb = new StringBuilder("<html><head>");
        String str = this.f235a;
        if (str == null) {
            C0184c c0184c = new C0184c("lateinit property appSessionId has not been initialized");
            String name = AbstractC0083a.class.getName();
            StackTraceElement[] stackTrace = c0184c.getStackTrace();
            int length = stackTrace.length;
            int i2 = -1;
            for (int i3 = 0; i3 < length; i3++) {
                if (name.equals(stackTrace[i3].getClassName())) {
                    i2 = i3;
                }
            }
            c0184c.setStackTrace((StackTraceElement[]) Arrays.copyOfRange(stackTrace, i2 + 1, length));
            throw c0184c;
        }
        sb.append("<script>const SECURE_TOKEN = '" + str + "';</script>");
        sb.append("</head><body><h1>Saved Notes</h1><p>*** Adding Notes is not possible at this moment. Please wait to recieve a license ***</p><hr><ul>");
        try {
            Cursor cursorQuery = getContentResolver().query(MemoProvider.f236b, null, null, null, null);
            if (cursorQuery != null) {
                while (cursorQuery.moveToNext()) {
                    try {
                        int columnIndex = cursorQuery.getColumnIndex("content");
                        if (columnIndex >= 0) {
                            String string2 = cursorQuery.getString(columnIndex);
                            sb.append("<li>");
                            sb.append(string2);
                            sb.append("</li>");
                        }
                    } finally {
                    }
                }
                cursorQuery.close();
            }
        } catch (Exception unused) {
            sb.append("<li>Error loading data.</li>");
        }
        sb.append("</ul></body></html>");
        webView.loadData(sb.toString(), "text/html", "UTF-8");
    }
}
```
This code is not really complicated. It is launcing a WebView (which is working with HTMP components and Javascript) and retrieving saved notes from MemoProvider content provider. but there is a key point in this code:
```Java
        webView.getSettings().setJavaScriptEnabled(true);
```
this code is allowing the the page to execute javascript codes. but what is so tricky about it? the tricky part is the code is using the notes content directly into page source without any specific encoding or sanitization. so there is a big risk of xss attack. but how to launch the attack?
let's take a look at our content provider MemoProvider:
<img width="1658" height="966" alt="image" src="https://github.com/user-attachments/assets/9dce0169-252e-4a22-b816-e51704380469" />
we can see that our content provider is using this URI:
content://com.athack.notebox.noteProvider/memos
and it is also loading notes from a local database called memories.db:
```
        this.f237a = new C0096b(getContext(), "memoirs.db", null, 1);
```

But what is C0096b?
<img width="1432" height="503" alt="image" src="https://github.com/user-attachments/assets/b76aa35a-1dfa-47ab-8c85-b5bcd7a220f8" />
we can see that the application is communicating with the local database using SQLite Queries. so there is a big chance that we could push notes to the application by communicating with exported activity and pushing notes into the database. let's see about that:
<img width="877" height="117" alt="image" src="https://github.com/user-attachments/assets/2d675310-bd4c-4bc7-b333-75e24d1d65dc" />
let's restart the application:
<img width="877" height="117" alt="image" src="https://github.com/user-attachments/assets/4c05b75d-a04c-476c-929f-0c58d450399c" />

we successfuly pushed a note into the application. now let's launch our attack. let's take a look at this code in MainActivity:
```Java
public final String validateSession(String str) {
            str.getClass();
            String str2 = MainActivity.this.f235a;
            if (str2 != null) {
                if (!str2.equals(str)) {
                    return "Error: 401 Unauthorized Session";
                }
                MainActivity mainActivity = MainActivity.this;
                mainActivity.getClass();
                return Vault.INSTANCE.open(mainActivity);
            }
            C0184c c0184c = new C0184c("lateinit property appSessionId has not been initialized");
            String name = AbstractC0083a.class.getName();
            StackTraceElement[] stackTrace = c0184c.getStackTrace();
            int length = stackTrace.length;
            int i2 = -1;
            for (int i3 = 0; i3 < length; i3++) {
                if (name.equals(stackTrace[i3].getClassName())) {
                    i2 = i3;
                }
            }
            c0184c.setStackTrace((StackTraceElement[]) Arrays.copyOfRange(stackTrace, i2 + 1, length));
            throw c0184c;
        }
```

as we can see, this code is a vulnerable spot to access the vault which is managing the security and encryption mechanisms. the code is checking a variable as a token to control the access to the Vault:
```Java
return Vault.INSTANCE.open(mainActivity);
```

<img width="2559" height="1385" alt="image" src="https://github.com/user-attachments/assets/bb1c6f71-8610-4b2d-a724-ad23a2368d0d" />

Where is the token? in the main activity, we can see a javascript code that is defining the token:
```Java
        sb.append("<script>const SECURE_TOKEN = '" + str + "';</script>");
```

so we have to pass it to validateSession to break into the vault. so our payload must be something like this:
```HTML
<img src=x onerror=alert(AndroidSys.validateSession(SECURE_TOKEN))>
```

let's push it into the database:
<img width="1234" height="127" alt="image" src="https://github.com/user-attachments/assets/fd7f929c-0de8-4576-a4b6-6b37649e8ca4" />

let's restart the application:
<img width="1080" height="2400" alt="Screenshot_1770694161" src="https://github.com/user-attachments/assets/43f513d6-79de-4e95-8e92-00388c7fe624" />
