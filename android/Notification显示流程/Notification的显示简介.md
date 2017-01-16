最近在使用Notification的时候遇到一个bug，`RemoteServiceException: Bad notification posted from package xxx, Couldn't expand RemoteViews for....`可把我玩坏了，而且不知是不是Instant Run的问题有些情况下可以正确运行，有时又不可以了，而且都是卸载安装，搞了大半天，google上也没找到合适的解决方案，主要应该出现在`Notification`的更新上，因为开始不更新是没问题的，所以有空再回来理一理这个坑

这个过程必定是通过binder实现的，先来理清各个角色

结论

- Notification#contentView不能为null

- Notification#[contentView|bigContentView|headsUpContentView].apply异常，实际上是`RemoteViews#apply`异常

# 流程

```java
NotificationManager.java

public void notify(int id, Notification notification)
{
    notify(null, id, notification);
}
```

```java
public void notify(String tag, int id, Notification notification)
{
    int[] idOut = new int[1];
    INotificationManager service = getService();
    String pkg = mContext.getPackageName();
    if (notification.sound != null) {
        notification.sound = notification.sound.getCanonicalUri();
        if (StrictMode.vmFileUriExposureEnabled()) {
            notification.sound.checkFileUriExposed("Notification.sound");
        }
    }
    fixLegacySmallIcon(notification, pkg);
    if (mContext.getApplicationInfo().targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
        if (notification.getSmallIcon() == null) {
            throw new IllegalArgumentException("Invalid notification (no valid small icon): " + notification);
        }
    }
    Notification stripped = notification.clone();
    Builder.stripForDelivery(stripped);
    try {
        service.enqueueNotificationWithTag(pkg, mContext.getOpPackageName(), tag, id, stripped, idOut, UserHandle.myUserId());
        if (id != idOut[0]) {
            Log.w(TAG, "notify: id corrupted: sent " + id + ", got back " + idOut[0]);
        }
    } catch (RemoteException e) {
    }
}
```

```java
@Override
public void enqueueNotificationWithTag(String pkg, String opPkg, String tag, int id,
        Notification notification, int[] idOut, int userId) throws RemoteException {
    enqueueNotificationInternal(pkg, opPkg, Binder.getCallingUid(),
            Binder.getCallingPid(), tag, id, notification, idOut, userId);
}
```

```java
void enqueueNotificationInternal(final String pkg, final String opPkg, final int callingUid,
        final int callingPid, final String tag, final int id, final Notification notification,
        int[] idOut, int incomingUserId) {
    if (DBG) {
        Slog.v(TAG, "enqueueNotificationInternal: pkg=" + pkg + " id=" + id
                + " notification=" + notification);
    }
    checkCallerIsSystemOrSameApp(pkg); //
    final boolean isSystemNotification = isUidSystem(callingUid) || ("android".equals(pkg));
    final boolean isNotificationFromListener = mListeners.isListenerPackage(pkg);

    final int userId = ActivityManager.handleIncomingUser(callingPid,
            callingUid, incomingUserId, true, false, "enqueueNotification", pkg);
    final UserHandle user = new UserHandle(userId);

    // Limit the number of notifications that any given package except the android
    // package or a registered listener can enqueue.  Prevents DOS attacks and deals with leaks.
    if (!isSystemNotification && !isNotificationFromListener) {
        //通知数量的限制，50条
    }

    if (pkg == null || notification == null) {
        throw new IllegalArgumentException("null not allowed: pkg=" + pkg
                + " id=" + id + " notification=" + notification);
    }

    if (notification.getSmallIcon() != null) {
        if (!notification.isValid()) {
            throw new IllegalArgumentException("Invalid notification (): pkg=" + pkg
                    + " id=" + id + " notification=" + notification);
        }
    }

    mHandler.post(new Runnable() {
        @Override
        public void run() {

            synchronized (mNotificationList) {

                // === Scoring ===

                // 0\. Sanitize inputs
                notification.priority = clamp(notification.priority, Notification.PRIORITY_MIN,
                        Notification.PRIORITY_MAX);
                // Migrate notification flags to scores
                if (0 != (notification.flags & Notification.FLAG_HIGH_PRIORITY)) {
                    if (notification.priority < Notification.PRIORITY_MAX) {
                        notification.priority = Notification.PRIORITY_MAX;
                    }
                } else if (SCORE_ONGOING_HIGHER &&
                        0 != (notification.flags & Notification.FLAG_ONGOING_EVENT)) {
                    if (notification.priority < Notification.PRIORITY_HIGH) {
                        notification.priority = Notification.PRIORITY_HIGH;
                    }
                }
                // force no heads up per package config
                if (!mRankingHelper.getPackagePeekable(pkg, callingUid)) {
                    if (notification.extras == null) {
                        notification.extras = new Bundle();
                    }
                    notification.extras.putInt(Notification.EXTRA_AS_HEADS_UP,
                            Notification.HEADS_UP_NEVER);
                }

                // 1\. initial score: buckets of 10, around the app [-20..20]
                final int score = notification.priority * NOTIFICATION_PRIORITY_MULTIPLIER;

                // 2\. extract ranking signals from the notification data
                final StatusBarNotification n = new StatusBarNotification(pkg, opPkg, id, tag, callingUid, callingPid, score, notification ,user);
                NotificationRecord r = new NotificationRecord(n, score);
                NotificationRecord old = mNotificationsByKey.get(n.getKey());
                if (old != null) { //更新通知的时候old不为null
                    // Retain ranking information from previous record
                    r.copyRankingInformation(old);
                }

                // Handle grouped notifications and bail out early if we can to avoid extracting signals.
                handleGroupedNotificationLocked(r, old, callingUid, callingPid);
                boolean ignoreNotification = removeUnusedGroupedNotificationLocked(r, old, callingUid, callingPid);
                //...

                if (ignoreNotification) {
                    return;
                }

                mRankingHelper.extractSignals(r);

                // 3\. Apply local rules

                // blocked apps
                if (ENABLE_BLOCKED_NOTIFICATIONS && !noteNotificationOp(pkg, callingUid)) {
                    if (!isSystemNotification) {
                        r.score = JUNK_SCORE;
                        Slog.e(TAG, "Suppressing notification from package " + pkg
                                + " by user request.");
                        mUsageStats.registerBlocked(r);
                    }
                }

                if (r.score < SCORE_DISPLAY_THRESHOLD) {
                    // Notification will be blocked because the score is too low.
                    return;
                }

                int index = indexOfNotificationLocked(n.getKey());
                if (index < 0) {
                    mNotificationList.add(r);
                    mUsageStats.registerPostedByApp(r);
                } else {
                    old = mNotificationList.get(index);
                    mNotificationList.set(index, r);
                    mUsageStats.registerUpdatedByApp(r, old);
                    // Make sure we don't lose the foreground service state.
                    notification.flags |=
                            old.getNotification().flags & Notification.FLAG_FOREGROUND_SERVICE;
                    r.isUpdate = true;
                }

                mNotificationsByKey.put(n.getKey(), r);

                // Ensure if this is a foreground service that the proper additional
                // flags are set.
                if ((notification.flags & Notification.FLAG_FOREGROUND_SERVICE) != 0) {
                    notification.flags |= Notification.FLAG_ONGOING_EVENT
                            | Notification.FLAG_NO_CLEAR;
                }

                applyZenModeLocked(r);
                mRankingHelper.sort(mNotificationList);

                if (notification.getSmallIcon() != null) {
                    StatusBarNotification oldSbn = (old != null) ? old.sbn : null;
                    mListeners.notifyPostedLocked(n, oldSbn);
                } else {
                    Slog.e(TAG, "Not posting notification without small icon: " + notification);
                    if (old != null && !old.isCanceled) {
                        mListeners.notifyRemovedLocked(n);
                    }
                    // ATTENTION: in a future release we will bail out here
                    // so that we do not play sounds, show lights, etc. for invalid
                    // notifications
                    Slog.e(TAG, "WARNING: In a future release this will crash the app: "
                            + n.getPackageName());
                }

                buzzBeepBlinkLocked(r);
            }
        }
    });

    idOut[0] = id;
}
```

## BaseStatusBar处理通知更新或发布

`BaseStatusBar`是一个抽象类，有以下两个方法来处理通知的添加或者更新

- addNotification （抽象方法，实现PhoneStatusBar）
- updateNotification(StatusBarNotification notification, RankingMap ranking)

### addNotification

```java
1166    @Override
1167    public void addNotification(StatusBarNotification notification, RankingMap ranking,
1168            Entry oldEntry) {
1169        if (DEBUG) Log.d(TAG, "addNotification key=" + notification.getKey());
1170
1171        Entry shadeEntry = createNotificationViews(notification);
1172        if (shadeEntry == null) {
1173            return;
1174        }
1175        boolean isHeadsUped = mUseHeadsUp && shouldInterrupt(shadeEntry);
1176        if (isHeadsUped) {
1177            mHeadsUpManager.showNotification(shadeEntry);
1178            // Mark as seen immediately
1179            setNotificationShown(notification);
1180        }
1181
1182        if (!isHeadsUped && notification.getNotification().fullScreenIntent != null) {
1183            // Stop screensaver if the notification has a full-screen intent.
1184            // (like an incoming phone call)
1185            awakenDreams();
1186
1187            // not immersive & a full-screen alert should be shown
1188            if (DEBUG) Log.d(TAG, "Notification has fullScreenIntent; sending fullScreenIntent");
1189            try {
1190                EventLog.writeEvent(EventLogTags.SYSUI_FULLSCREEN_NOTIFICATION,
1191                        notification.getKey());
1192                notification.getNotification().fullScreenIntent.send();
1193                shadeEntry.notifyFullScreenIntentLaunched();
1194                MetricsLogger.count(mContext, "note_fullscreen", 1);
1195            } catch (PendingIntent.CanceledException e) {
1196            }
1197        }
1198        addNotificationViews(shadeEntry, ranking);
1199        // Recalculate the position of the sliding windows and the titles.
1200        setAreThereNotifications();
1201    }
```

### 处理更新

```java
1828    public void More ...updateNotification(StatusBarNotification notification, RankingMap ranking) {
1829        if (DEBUG) Log.d(TAG, "updateNotification(" + notification + ")");
1830
1831        final String key = notification.getKey();
1832        boolean wasHeadsUp = isHeadsUp(key);
1833        Entry oldEntry;
1834        if (wasHeadsUp) {
1835            oldEntry = mHeadsUpNotificationView.getEntry();
1836        } else {
1837            oldEntry = mNotificationData.get(key);
1838        }
1839        if (oldEntry == null) {
1840            return;
1841        }
1842
1843        final StatusBarNotification oldNotification = oldEntry.notification;
1844
1845        // XXX: modify when we do something more intelligent with the two content views
1846        final RemoteViews oldContentView = oldNotification.getNotification().contentView;
1847        Notification n = notification.getNotification();
1848        final RemoteViews contentView = n.contentView;
1849        final RemoteViews oldBigContentView = oldNotification.getNotification().bigContentView;
1850        final RemoteViews bigContentView = n.bigContentView;
1851        final RemoteViews oldHeadsUpContentView = oldNotification.getNotification().headsUpContentView;
1852        final RemoteViews headsUpContentView = n.headsUpContentView;
1853        final Notification oldPublicNotification = oldNotification.getNotification().publicVersion;
1854        final RemoteViews oldPublicContentView = oldPublicNotification != null
1855                ? oldPublicNotification.contentView : null;
1856        final Notification publicNotification = n.publicVersion;
1857        final RemoteViews publicContentView = publicNotification != null
1858                ? publicNotification.contentView : null;
1859
1860        if (DEBUG) {
1861            Log.d(TAG, "old notification: when=" + oldNotification.getNotification().when
1862                    + " ongoing=" + oldNotification.isOngoing()
1863                    + " expanded=" + oldEntry.expanded
1864                    + " contentView=" + oldContentView
1865                    + " bigContentView=" + oldBigContentView
1866                    + " publicView=" + oldPublicContentView
1867                    + " rowParent=" + oldEntry.row.getParent());
1868            Log.d(TAG, "new notification: when=" + n.when
1869                    + " ongoing=" + oldNotification.isOngoing()
1870                    + " contentView=" + contentView
1871                    + " bigContentView=" + bigContentView
1872                    + " publicView=" + publicContentView);
1873        }
1874
1875        // Can we just reapply the RemoteViews in place?
1876
1877        // 1U is never null
1878        boolean contentsUnchanged = oldEntry.expanded != null
1879                && contentView.getPackage() != null
1880                && oldContentView.getPackage() != null
1881                && oldContentView.getPackage().equals(contentView.getPackage())
1882                && oldContentView.getLayoutId() == contentView.getLayoutId();
1883        // large view may be null
1884        boolean bigContentsUnchanged =
1885                (oldEntry.getBigContentView() == null && bigContentView == null)
1886                || ((oldEntry.getBigContentView() != null && bigContentView != null)
1887                    && bigContentView.getPackage() != null
1888                    && oldBigContentView.getPackage() != null
1889                    && oldBigContentView.getPackage().equals(bigContentView.getPackage())
1890                    && oldBigContentView.getLayoutId() == bigContentView.getLayoutId());
1891        boolean headsUpContentsUnchanged =
1892                (oldHeadsUpContentView == null && headsUpContentView == null)
1893                || ((oldHeadsUpContentView != null && headsUpContentView != null)
1894                    && headsUpContentView.getPackage() != null
1895                    && oldHeadsUpContentView.getPackage() != null
1896                    && oldHeadsUpContentView.getPackage().equals(headsUpContentView.getPackage())
1897                    && oldHeadsUpContentView.getLayoutId() == headsUpContentView.getLayoutId());
1898        boolean publicUnchanged  =
1899                (oldPublicContentView == null && publicContentView == null)
1900                || ((oldPublicContentView != null && publicContentView != null)
1901                        && publicContentView.getPackage() != null
1902                        && oldPublicContentView.getPackage() != null
1903                        && oldPublicContentView.getPackage().equals(publicContentView.getPackage())
1904                        && oldPublicContentView.getLayoutId() == publicContentView.getLayoutId());
1905        boolean updateTicker = n.tickerText != null
1906                && !TextUtils.equals(n.tickerText,
1907                oldEntry.notification.getNotification().tickerText);
1908
1909        final boolean shouldInterrupt = shouldInterrupt(notification);
1910        final boolean alertAgain = alertAgain(oldEntry, n);
1911        boolean updateSuccessful = false;
1912        if (contentsUnchanged && bigContentsUnchanged && headsUpContentsUnchanged
1913                && publicUnchanged) {
1914            if (DEBUG) Log.d(TAG, "reusing notification for key: " + key);
1915            oldEntry.notification = notification;
1916            try {
1917                if (oldEntry.icon != null) {
1918                    // Update the icon
1919                    final StatusBarIcon ic = new StatusBarIcon(notification.getPackageName(),
1920                            notification.getUser(),
1921                            n.icon,
1922                            n.iconLevel,
1923                            n.number,
1924                            n.tickerText);
1925                    oldEntry.icon.setNotification(n);
1926                    if (!oldEntry.icon.set(ic)) {
1927                        handleNotificationError(notification, "Couldn't update icon: " + ic);
1928                        return;
1929                    }
1930                }
1931
1932                if (wasHeadsUp) {
1933                    if (shouldInterrupt) {
1934                        updateHeadsUpViews(oldEntry, notification);
1935                        if (alertAgain) {
1936                            resetHeadsUpDecayTimer();
1937                        }
1938                    } else {
1939                        // we updated the notification above, so release to build a new shade entry
1940                        mHeadsUpNotificationView.releaseAndClose();
1941                        return;
1942                    }
1943                } else {
1944                    if (shouldInterrupt && alertAgain) {
1945                        removeNotificationViews(key, ranking);
1946                        addNotification(notification, ranking);  //this will pop the headsup
1947                    } else {
1948                        updateNotificationViews(oldEntry, notification);
1949                    }
1950                }
1951                mNotificationData.updateRanking(ranking);
1952                updateNotifications();
1953                updateSuccessful = true;
1954            }
1955            catch (RuntimeException e) {
1956                // It failed to add cleanly.  Log, and remove the view from the panel.
1957                Log.w(TAG, "Couldn't reapply views for package " + contentView.getPackage(), e);
1958            }
1959        }
1960        if (!updateSuccessful) {
1961            if (DEBUG) Log.d(TAG, "not reusing notification for key: " + key);
1962            if (wasHeadsUp) {
1963                if (shouldInterrupt) {
1964                    if (DEBUG) Log.d(TAG, "rebuilding heads up for key: " + key);
1965                    Entry newEntry = new Entry(notification, null);
1966                    ViewGroup holder = mHeadsUpNotificationView.getHolder();
1967                    if (inflateViewsForHeadsUp(newEntry, holder)) {
1968                        mHeadsUpNotificationView.showNotification(newEntry);
1969                        if (alertAgain) {
1970                            resetHeadsUpDecayTimer();
1971                        }
1972                    } else {
1973                        Log.w(TAG, "Couldn't create new updated headsup for package "
1974                                + contentView.getPackage());
1975                    }
1976                } else {
1977                    if (DEBUG) Log.d(TAG, "releasing heads up for key: " + key);
1978                    oldEntry.notification = notification;
1979                    mHeadsUpNotificationView.releaseAndClose();
1980                    return;
1981                }
1982            } else {
1983                if (shouldInterrupt && alertAgain) {
1984                    if (DEBUG) Log.d(TAG, "reposting to invoke heads up for key: " + key);
1985                    removeNotificationViews(key, ranking);
1986                    addNotification(notification, ranking);  //this will pop the headsup
1987                } else {
1988                    if (DEBUG) Log.d(TAG, "rebuilding update in place for key: " + key);
1989                    oldEntry.notification = notification;
1990                    final StatusBarIcon ic = new StatusBarIcon(notification.getPackageName(),
1991                            notification.getUser(),
1992                            n.icon,
1993                            n.iconLevel,
1994                            n.number,
1995                            n.tickerText);
1996                    oldEntry.icon.setNotification(n);
1997                    oldEntry.icon.set(ic);
1998                    inflateViews(oldEntry, mStackScroller, wasHeadsUp);
1999                    mNotificationData.updateRanking(ranking);
2000                    updateNotifications();
2001                }
2002            }
2003        }
2004
2005        // Update the veto button accordingly (and as a result, whether this row is
2006        // swipe-dismissable)
2007        updateNotificationVetoButton(oldEntry.row, notification);
2008
2009        // Is this for you?
2010        boolean isForCurrentUser = isNotificationForCurrentProfiles(notification);
2011        if (DEBUG) Log.d(TAG, "notification is " + (isForCurrentUser ? "" : "not ") + "for you");
2012
2013        // Restart the ticker if it's still running
2014        if (updateTicker && isForCurrentUser) {
2015            haltTicker();
2016            tick(notification, false);
2017        }
2018
2019        // Recalculate the position of the sliding windows and the titles.
2020        setAreThereNotifications();
2021        updateExpandedViewPos(EXPANDED_LEAVE_ALONE);
2022    }
2023
```

### RemoteView#apply

```java
public View apply(Context context, ViewGroup parent, OnClickHandler handler) {
    RemoteViews rvToApply = getRemoteViewsToApply(context);

    View result;
    // RemoteViews may be built by an application installed in another
    // user. So build a context that loads resources from that user but
    // still returns the current users userId so settings like data / time formats
    // are loaded without requiring cross user persmissions.
    final Context contextForResources = getContextForResources(context);
    Context inflationContext = new ContextWrapper(context) {
        @Override
        public Resources getResources() {
            return contextForResources.getResources();
        }
        @Override
        public Resources.Theme getTheme() {
            return contextForResources.getTheme();
        }
        @Override
        public String getPackageName() {
            return contextForResources.getPackageName();
        }
    };

    LayoutInflater inflater = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

    // Clone inflater so we load resources from correct context and
    // we don't add a filter to the static version returned by getSystemService.
    inflater = inflater.cloneInContext(inflationContext);
    inflater.setFilter(this);
    result = inflater.inflate(rvToApply.getLayoutId(), parent, false);

    rvToApply.performApply(result, parent, handler);

    return result;
}
```

## 参考

[NotificationManagerService笔记](http://blog.csdn.net/guoqifa29/article/details/44133905)

[Android NotificationListenerService原理简介](http://blog.csdn.net/yueqian_scut/article/details/51417547)

[Heads-Up Notification](http://blog.csdn.net/wds1181977/article/details/49783787)

[Android dev Notification](https://developer.android.com/guide/topics/ui/notifiers/notifications.html)
