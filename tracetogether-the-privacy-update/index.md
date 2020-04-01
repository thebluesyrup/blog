# TraceTogether: the privacy update

> Author's note: In this article, we will only be exploring the iOS app. Due to my inexperience with Android apps, I have refrained from discussing about it. I also personally thank the TraceTogether Team for realising the idea in such a short timeframe.
>
> This article was written to tackle a specific, microscopic issue with the TraceTogether app and is not intended to discredit the TraceTogether initiative. TraceTogether was created with good intentions and this article is meant to provide my findings and opinions on a very specific matter.
>
> These issues do happen in software development and TraceTogether is no exception. The purpose of the TraceTogether app makes this a more sensitive issue than under normal circumstances and therefore should be treated as such.

On 31st March 2020, GovTech released an update for TraceTogether iOS app.

![TraceTogether version 1.4 changelog on Apple AppStore](./media/tracetogether-apple-appstore-changelog.png)

It mentions "Performance improvements" but not anything related to analytics. So, have they finally removed the analytics? Curious, I download the app to see what's changed.

I used [Telerik Fiddler 4](https://www.telerik.com/download/fiddler) as a network proxy to sniff the traffic from my iPhone and get information on the web requests sent by the TraceTogether app. Through that, I opened the app, freshly downloaded from the AppStore, and went through the sign-up process. This is what came up:

![TraceTogether network logs](./media/tracetogether-network-logs.png)

The first thing I noticed is there were no more requests to snowplow-mobile.wogaa.gov.sg. This is great news! This means that the TraceTogether Team have reverted the accidental inclusion of WOGAA Snowplow.

However, the keen-eyed may have noticed another set suspiciously analytics-like URLs:

- https://app-measurement.com/a
- https://firebaselogging-pa.googleapis.com/v1/firelog/legacy/batchlog

Both of these services are Google-owned analytics platforms.

The TraceTogether app uses Firebase (a Google brand) for SMS OTP authentication. To make it easier to interact with the Firebase API, they've opted to use the Firebase SDK. This provides convenience functions to more quickly-integrate the Firebase services into the app. This however also meant that certain telemtry would be collected by default. This results in the unwatned performance analytics that we're going to look into.

Upon closer inspection, we notice these data being sent:

<details>
<summary>Data sent to app-measurement.com</summary>

```
POST https://app-measurement.com/a HTTP/1.1
Host: app-measurement.com
Content-Type: application/x-www-form-urlencoded
Connection: keep-alive
Accept: */*
Accept-Language: en-sg
Content-Length: 1151
Accept-Encoding: gzip, deflate, br
User-Agent: TraceTogether/23 CFNetwork/1121.2.2 Darwin/19.3.0


 p


_oauto

_r

_uwa

_c

_pfo
"
_scInitialScreenViewController

_si ҃     _f݉   .݉   ._fi ݉   ._fot   ̢ .      .(݉   .0݉   .BiosJ13.3.1R
iPhone10,4Zen-sg` rsg.gov.tech.bluetrace 1.4        293E617885B149279E485D7CDCF6C1BC  )1:510731179958:ios:28487b80a62459446d5317 $DFCF2026-B857-4889-B915-FF77B702851D cFaJbOn7n0YshvmU1BVOg3         
 /

_sid Ӑ 

_sno


_oauto_s݉   .U

_si ҃     

_et
"
_scInitialScreenViewController


_oauto_e݉   .M

_si ҃     
"
_scInitialScreenViewController


_oauto_vs     . 

_si ҃     
"
_pcInitialScreenViewController
$
_scOnboardingStep1ViewController


_oauto

_pi ҃     _vs     .      .݉   ._fi ݉   ._fot   ̢ .݉   ._sid  Ӑ      ._sno      ._lte      ._se       .(݉   .0     .8݉   .BiosJ13.3.1R
iPhone10,4Zen-sg` rsg.gov.tech.bluetrace 1.4        293E617885B149279E485D7CDCF6C1BC  )1:510731179958:ios:28487b80a62459446d5317 ݉   . $DFCF2026-B857-4889-B915-FF77B702851D cFaJbOn7n0YshvmU1BVOg3         
```

</details>

<details>
<summary>Data sent to firebaselogging-pa.googleapis.com</summary>

```
POST https://firebaselogging-pa.googleapis.com/v1/firelog/legacy/batchlog HTTP/1.1
Host: firebaselogging-pa.googleapis.com
Content-Type: application/x-protobuf
Accept-Language: en-sg
Connection: keep-alive
Accept: */*
User-Agent: datatransport/4.0.0 fllsupport/1.4.0 apple/
X-Goog-Api-Key: AIzaSyB4G40IhwotnuD26vAV-0PAxOVr5zn1BCc
Accept-Encoding: gzip
Content-Length: 891


 
;"713"13.3.1*232SG:iPhoneBenZsg.gov.tech.bluetrace       .2 : J
iPhone10,4R)1:510731179958:ios:28487b80a62459446d5317bsg.gov.tech.bluetrace  60602000  13.3.1 13.0    
11E146-17E255     apple-sdk/17E255 fire-abt/3.1.2 fire-analytics/6.2.2 fire-auth/6.4.3 fire-fun/2.5.1 fire-iid/4.3.1 fire-install/1.1.0 fire-ios/6.6.2 fire-rc/4.4.7 fire-str/3.5.0 firebase-crashlytics/4.0.0-beta.4 swift/true xcode/11E146  x    ܾ  ό   .2 : J
iPhone10,4R)1:510731179958:ios:28487b80a62459446d5317bsg.gov.tech.bluetrace  60602000   13.3.1 13.0    
11E146-17E255     apple-sdk/17E255 fire-abt/3.1.2 fire-analytics/6.2.2 fire-auth/6.4.3 fire-fun/2.5.1 fire-iid/4.3.1 fire-install/1.1.0 fire-ios/6.6.2 fire-rc/4.4.7 fire-str/3.5.0 firebase-crashlytics/4.0.0-beta.4 swift/true xcode/11E146  x            .@    
```

</details>

Unfortunately, [we can't easily decode this information](https://stackoverflow.com/questions/54461349/how-to-decrypt-firebase-requests-to-app-measurement-com/54463682). But we can still infer from the readable sections what kind of data is being sent.

Furthermore, we can look at Google's documentation on data that is automatically collected:

- https://support.google.com/firebase/answer/6317485?hl=en&ref_topic=6317484
- https://support.google.com/firebase/answer/6317486?hl=en&ref_topic=6317484

![Firebase automatically-collected events](./media/tracetogether-firebase-auto-events.png)

![Firebae automatically-collected user info](./media/tracetogether-firebase-auto-user-info.png)

**Is everything mentioned in the docs collected? No.** Things such as in-app purchase, subscriptions and advertisement-related analytics won't be sent as those events will never happen.

**But that doesn't mean no analytics is collected without consent.**

From [Google's documentation](https://firebase.google.com/docs/analytics/configure-data-collection#permanently_deactivate_collection_2), Firebase analytics can be disabled with a single line in the `Info.plist` file:

```xml
<key>FIREBASE_ANALYTICS_COLLECTION_DEACTIVATED</key>
<true/>
```

> Note: `Info.plist` is file where app-wide configuration (such as requested permissions) are stored.

With this in mind, it seems like this was an honest oversight from the team. I've already enquired about the Firebase analytics a week prior, so they may have forgotten to disable it.

## Conclusion

Overall, while removing WOGAA Snowplow is a good step in the right direction, there are still 2 glaring issues that weren't solved by the latest update:

1. Transparency

    Other than via email requests or similar mediums, the TraceTogether team have not publicly acknowledged (on their website, social media or changelogs) that they collected data from their users.

    Furthermore, they have not stated how they would handle data that has already been collected.

    Even though this the inclusion of the analytics platform was accidental, they should have been more transparent about their mistake and take the opportunity to reassure their users that the issue has been resolved.

2. Firebase analytics

    While removing WOGAA Snowplow is a great step to ensuring user privacy, they have neglected to disable Firebase analytics. This seems to be a missed opportunity to sort out the analytics problem as a whole.

Overall, the stance of this article stems from the idea that, if a company who has to abide with strict privacy laws such as GDPR can be transparent, it is  fair to assume that the technologically-progressive government can do  the same, if not better.

> Author's note: I've personally contacted the TraceTogether Team via email and have thanked them for working around the clock to making this app possible. I wish nothing but the best for the TraceTogether Team and their future endeavors and hope that this can be taken as a learning opportunity to improve their processes and be more transparent to their users.
