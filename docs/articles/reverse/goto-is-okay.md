# GOTO is Okay

**Date of publication:** 04 Mar 2026

## Case description

I downloaded an application from the Google Play Store. When the app starts there is an ads window with timer for closing window and return to normal using the app. The main task is cutting the window and restore the client-oriented user-flow.

## Legal disclaimer

Based on Article 6 of DIRECTIVE 2009/24/EC ([link to source](https://eur-lex.europa.eu/eli/dir/2009/24/oj/eng)) and similar law acts in other countries.

The float window of ads is not interoperability with a software in fundamental. Due to the fact that third-party software (Google Mobile Ads SDK) prevents me from using the software I legally downloaded from the Google Play Store. 

By forcing time-based lock on the UI, the third-party SDK prevents the software from performing its primary task on the user's device.

## Solution

To resolve this, we target `AdActivity` responsible for rendering interception. Move through the `onCreate` method from the start to the end of method with `goto` for avoiding all inner conditions and logic `smali/com/google/android/gms/ads/AdActivity.smali`

```smali
.method protected final onCreate(Landroid/os/Bundle;)V
    .locals 2

    .line 1
    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V
    
    goto :exit_point #add this pointer

    #some code

    .line 32
    .line 33
    .line 34

    :exit_point #add this mark

    invoke-virtual {p0}, Landroid/app/Activity;->finish()V

    .line 35
    .line 36
    .line 37
    return-void
.end method
```

### Result

```txt
// 0. The system starts collecting the transition for AdActivity
03-04 01:22:43.371  V WindowManager: Collecting transition: ActivityRecord{... AdActivity}

// 1. The Activity gets focus
03-04 01:22:43.401  D InputDispatcher: Focused application: ActivityRecord{... AdActivity}

// 2. Our GOTO hits finish() (f - flag)
03-04 01:22:43.435  V WindowManager: Collecting transition (FINISHING): ActivityRecord{... AdActivity f}}
03-04 01:22:43.435  D ActivityTaskManager: scheduleTopResumedActivityChanged, onTop=false, r=ActivityRecord{... AdActivity f}}

// 3. Complete destruction of the surface layer
03-04 01:22:43.533  I Layer: id=27875 Destroyed ActivityRecord{... AdActivity}
```

### Timing

- 64ms - from `super()` to `finish()` methods
- 162ms - for the whole cycle

## Summary

Thus, the using the unconditional jump operator `goto` is the minimal action for "necessary in order to achieve interoperability" (Article 6, 1(c)). 

---

Thanks for reading. I hope you enjoyed it. :)
