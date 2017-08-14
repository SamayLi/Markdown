android 多用户管理UserManager
==============================

一.概述
-----------
Android从4.2开始支持多用户模式，不同的用户运行在不同的用户空间，相关的系统设置是各不相同而且不同用户安装的应用和应用数据也是不一样的，但是系统中和硬件相关的设计则是共用的。

Android的多用户实现主要通过UserManagerService(UMS)对多用户进行创建、删除等操作。

*Binder服务端:UMS继承于IUserManager.Stub，作为Binder服务端；

*Binder服务器:UserManager的成员变量mService继承于IUserManager.Stub.Proxy,该变量作为binder客户端。采用标准的binder通信进行信息传递

二.概念
------------

###2.1   userId,uid,appId

userId:用户id; appId:跟用户空间无关的应用程序id;取值范围0<=appId<100000;uid:跟用户空间紧密相关的应用程序id.

```java
    uid = userId * 100000 + appId
```

###2.2   UserHandler

常见方法:

方法|描述
----|----
isSameUser|比较两个uid的userId是否相同
isSameApp|比较两个uid的appId是否相同
isApp|appId是否属于区间[10000,19999]
isIsolated|appId是否属于区间[99000,99999]
getIdentifier|获取UserHandler所对应的userId

常见成员变量:

userId|赋值|含义
----|----|----
USER_OWNER|0|拥有者
USER_ALL|-1|所有用户
USER_CURRENT|-2|当前活动用户
USER_CURRENT_OR_SELF|-3|当前用户或者调用者所在用户
USER_NULL|-1000|未定义用户

###2.3  UserInfo

UserInfo代表的是一个用户的信息,涉及到的flags及其含义,如下:

flags|含义
-----|-----
FLAG_PRIMARY|主用户，只有一个user具有该标识
FLAG_ADMIN|具有管理特权的用户，例如创建或删除其他用户
FLAG_GUEST|访客用户，可能是临时的
FLAG_RESTRICTED|限制性用户，较普通用户具有更多限制，例如禁止安装app或者管理wifi等
FLAG_INITIALIZED|表明用户已初始化
FLAG_MANAGED_PROFILE|表明该用户是另一个用户的轮廓
FLAG_DISABLED|表明该用户处于不可用状态

###2.4  UserState

```java
        // User is first coming up.
        public final static int STATE_BOOTING = 0;
        // User is in the locked state.
        public final static int STATE_RUNNING_LOCKED = 1;
        // User is in the unlocking state.
        public final static int STATE_RUNNING_UNLOCKING = 2;
        // User is in the running state.
        public final static int STATE_RUNNING_UNLOCKED = 3;
        // User is in the initial process of being stopped.
        public final static int STATE_STOPPING = 4;
        // User is in the final phase of stopping, sending Intent.ACTION_SHUTDOWN.
        public final static int STATE_SHUTDOWN = 5;
```

三.UMS的启动

UMS在PMS中初始化
```java
    public PackageManagerService(Context context, Installer installer,
                boolean factoryTest, boolean onlyCore) {
            ......
                 sUserManager = new UserManagerService(context, this, mPackages);
            .....
                }
```
UMS的初始化操作
```java
    private UserManagerService(Context context, PackageManagerService pm,
                Object packagesLock, File dataDir) {
            mContext = context;
            mPm = pm;
            mPackagesLock = packagesLock;
            mHandler = new MainHandler();
            synchronized (mPackagesLock) {
                //创建目录/data/system/users
                mUsersDir = new File(dataDir, USER_INFO_DIR);
                mUsersDir.mkdirs();
                //创建目录/data/system/users/0
                File userZeroDir = new File(mUsersDir, String.valueOf(UserHandle.USER_SYSTEM));
                userZeroDir.mkdirs();
                FileUtils.setPermissions(mUsersDir.toString(),
                        FileUtils.S_IRWXU | FileUtils.S_IRWXG | FileUtils.S_IROTH | FileUtils.S_IXOTH,
                        -1, -1);
                //mYserListFile文件路径为/data/system/users/userlist.xml
                mUserListFile = new File(mUsersDir, USER_LIST_FILENAME);
                initDefaultGuestRestrictions();
                //解析userlist.xml文件
                readUserListLP();
                sInstance = this;
            }
            mLocalService = new LocalService();
            LocalServices.addService(UserManagerInternal.class, mLocalService);
            mLockPatternUtils = new LockPatternUtils(mContext);
            //表示System正处于启动阶段
            mUserStates.put(UserHandle.USER_SYSTEM, UserState.STATE_BOOTING);
        }
```

四.创建一个用户
---------------
创建一个多用户主要是在UMS中createUserInternalUnchecked中执行
```java
    private UserInfo createUserInternalUnchecked(String name, int flags, int parentId) {
        //检查一下手机是否处于低内存设备
        if (ActivityManager.isLowRamDeviceStatic()) {
            return null;
        }
        //判断需要创建的user是否是guest用户
        final boolean isGuest = (flags & UserInfo.FLAG_GUEST) != 0;
        //判断需要创建的user是否是profile用户
        final boolean isManagedProfile = (flags & UserInfo.FLAG_MANAGED_PROFILE) != 0;
        //判断需要创建的user是否是restricted用户
        final boolean isRestricted = (flags & UserInfo.FLAG_RESTRICTED) != 0;
        final boolean isDemo = (flags & UserInfo.FLAG_DEMO) != 0;
        final long ident = Binder.clearCallingIdentity();
        UserInfo userInfo;
        UserData userData;
        final int userId;
        try {
            synchronized (mPackagesLock) {
                UserData parent = null;
                if (parentId != UserHandle.USER_NULL) {
                    synchronized (mUsersLock) {
                        parent = getUserDataLU(parentId);
                    }
                    if (parent == null) return null;
                }
                if (isManagedProfile && !canAddMoreManagedProfiles(parentId, false)) {
                    Log.e(LOG_TAG, "Cannot add more managed profiles for user " + parentId);
                    return null;
                }
                if (!isGuest && !isManagedProfile && !isDemo && isUserLimitReached()) {
                    // If we're not adding a guest/demo user or a managed profile and the limit has
                    // been reached, cannot add a user.
                    return null;
                }
                // If we're adding a guest and there already exists one, bail.
                if (isGuest && findCurrentGuestUser() != null) {
                    return null;
                }
                // In legacy mode, restricted profile's parent can only be the owner user
                if (isRestricted && !UserManager.isSplitSystemUser()
                        && (parentId != UserHandle.USER_SYSTEM)) {
                    Log.w(LOG_TAG, "Cannot add restricted profile - parent user must be owner");
                    return null;
                }
                if (isRestricted && UserManager.isSplitSystemUser()) {
                    if (parent == null) {
                        Log.w(LOG_TAG, "Cannot add restricted profile - parent user must be "
                                + "specified");
                        return null;
                    }
                    if (!parent.info.canHaveProfile()) {
                        Log.w(LOG_TAG, "Cannot add restricted profile - profiles cannot be "
                                + "created for the specified parent user id " + parentId);
                        return null;
                    }
                }
                if (!UserManager.isSplitSystemUser() && (flags & UserInfo.FLAG_EPHEMERAL) != 0
                        && (flags & UserInfo.FLAG_DEMO) == 0) {
                    Log.e(LOG_TAG,
                            "Ephemeral users are supported on split-system-user systems only.");
                    return null;
                }
                // In split system user mode, we assign the first human user the primary flag.
                // And if there is no device owner, we also assign the admin flag to primary user.
                if (UserManager.isSplitSystemUser()
                        && !isGuest && !isManagedProfile && getPrimaryUser() == null) {
                    flags |= UserInfo.FLAG_PRIMARY;
                    synchronized (mUsersLock) {
                        if (!mIsDeviceManaged) {
                            flags |= UserInfo.FLAG_ADMIN;
                        }
                    }
                }
                //得到新用户的userId
                userId = getNextAvailableId();
                //创建data/system/users/{usdId}
                Environment.getUserSystemDirectory(userId).mkdirs();
                boolean ephemeralGuests = Resources.getSystem()
                        .getBoolean(com.android.internal.R.bool.config_guestUserEphemeral);

                synchronized (mUsersLock) {
                    // Add ephemeral flag to guests/users if required. Also inherit it from parent.
                    if ((isGuest && ephemeralGuests) || mForceEphemeralUsers
                            || (parent != null && parent.info.isEphemeral())) {
                        flags |= UserInfo.FLAG_EPHEMERAL;
                    }
                    //为该新用户创建UserInfo
                    userInfo = new UserInfo(userId, name, null, flags);
                    //设置序列号
                    userInfo.serialNumber = mNextSerialNumber++;
                    //设置创建时间
                    long now = System.currentTimeMillis();
                    userInfo.creationTime = (now > EPOCH_PLUS_30_YEARS) ? now : 0;
                    //设置partial,表示正在创建
                    userInfo.partial = true;
                    userInfo.lastLoggedInFingerprint = Build.FINGERPRINT;
                    userData = new UserData();
                    userData.info = userInfo;
                    mUsers.put(userId, userData);
                }
                //将新建的用户信息写入到/data/system/users/{userId}/{userId}.xml
                writeUserLP(userData);
                //将用户信息写入到userlist.xml
                writeUserListLP();
                if (parent != null) {
                    if (isManagedProfile) {
                        if (parent.info.profileGroupId == UserInfo.NO_PROFILE_GROUP_ID) {
                            parent.info.profileGroupId = parent.info.id;
                            writeUserLP(parent);
                        }
                        userInfo.profileGroupId = parent.info.profileGroupId;
                    } else if (isRestricted) {
                        if (parent.info.restrictedProfileParentId == UserInfo.NO_PROFILE_GROUP_ID) {
                            parent.info.restrictedProfileParentId = parent.info.id;
                            writeUserLP(parent);
                        }
                        userInfo.restrictedProfileParentId = parent.info.restrictedProfileParentId;
                    }
                }
            }
            final StorageManager storage = mContext.getSystemService(StorageManager.class);
            storage.createUserKey(userId, userInfo.serialNumber, userInfo.isEphemeral());
            //为新建的user准备存储区域
            mPm.prepareUserData(userId, userInfo.serialNumber,
                    StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
            //给上步为新建user准备的存储区域写入数据
            mPm.createNewUser(userId);
            //设置partial，表示创建结束
            userInfo.partial = false;
            synchronized (mPackagesLock) {
                //更新/data/system/users/{userId}/{userId}.xml
                writeUserLP(userData);
            }
            //更新mUserIds数据组
            updateUserIds();
            Bundle restrictions = new Bundle();
            if (isGuest) {
                synchronized (mGuestRestrictions) {
                    restrictions.putAll(mGuestRestrictions);
                }
            }
            synchronized (mRestrictionsLock) {
                mBaseUserRestrictions.append(userId, restrictions);
            }
            //给新创建的用户赋予默认权限
            mPm.onNewUserCreated(userId);
            //发送用户创建完成的广播
            Intent addedIntent = new Intent(Intent.ACTION_USER_ADDED);
            addedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
            mContext.sendBroadcastAsUser(addedIntent, UserHandle.ALL,
                    android.Manifest.permission.MANAGE_USERS);
            MetricsLogger.count(mContext, isGuest ? TRON_GUEST_CREATED : TRON_USER_CREATED, 1);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        return userInfo;
    }
```
五.切换用户
-----------
用户切换用户通过调用AMS中switchUser方法实现
```java
    public boolean switchUser(final int targetUserId) {
            enforceShellRestriction(UserManager.DISALLOW_DEBUGGING_FEATURES, targetUserId);
            UserInfo currentUserInfo;
            UserInfo targetUserInfo;
            synchronized (this) {
                //获得当前用户的UserInfo
                int currentUserId = mUserController.getCurrentUserIdLocked();
                currentUserInfo = mUserController.getUserInfo(currentUserId);
                //获得目标用户的UserInfo
                targetUserInfo = mUserController.getUserInfo(targetUserId);
                if (targetUserInfo == null) {
                    Slog.w(TAG, "No user info for user #" + targetUserId);
                    return false;
                }
                if (!targetUserInfo.isDemo() && UserManager.isDeviceInDemoMode(mContext)) {
                    Slog.w(TAG, "Cannot switch to non-demo user #" + targetUserId
                            + " when device is in demo mode");
                    return false;
                }
                if (!targetUserInfo.supportsSwitchTo()) {
                    Slog.w(TAG, "Cannot switch to User #" + targetUserId + ": not supported");
                    return false;
                }
                //如果用户是Profile则不允许使用switchUser方法切换用户
                if (targetUserInfo.isManagedProfile()) {
                    Slog.w(TAG, "Cannot switch to User #" + targetUserId + ": not a full user");
                    return false;
                }
                mUserController.setTargetUserIdLocked(targetUserId);
            }
            //发送切换用户的消息
            Pair<UserInfo, UserInfo> userNames = new Pair<>(currentUserInfo, targetUserInfo);
            mUiHandler.removeMessages(START_USER_SWITCH_UI_MSG);
            mUiHandler.sendMessage(mUiHandler.obtainMessage(START_USER_SWITCH_UI_MSG, userNames));
            return true;
        }
```

mUiHandler收到START_USER_SWITCH_UI_MSG消息后的处理,显示一个Dialog
```java
     case START_USER_SWITCH_UI_MSG: {
                    mUserController.showUserSwitchDialog((Pair<UserInfo, UserInfo>) msg.obj);
                    break;
                }
```

这个Dialog显示之后会调用startUser这个方法
```java
     void startUser() {
            synchronized (this) {
                if (!mStartedUser) {
                    mService.mUserController.startUserInForeground(mUserId, this);
                    mStartedUser = true;
                    final View decorView = getWindow().getDecorView();
                    if (decorView != null) {
                        decorView.getViewTreeObserver().removeOnWindowShownListener(this);
                    }
                    mHandler.removeMessages(MSG_START_USER);
                }
            }
        }
```
上述方法本质上调用的是UserController中的startUser方法，而这个方法本质上是启动多用户的最关键的方法
```java
    boolean startUser(final int userId, final boolean foreground) {
        if (mService.checkCallingPermission(INTERACT_ACROSS_USERS_FULL)
                != PackageManager.PERMISSION_GRANTED) {
            String msg = "Permission Denial: switchUser() from pid="
                    + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid()
                    + " requires " + INTERACT_ACROSS_USERS_FULL;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }

        Slog.i(TAG, "Starting userid:" + userId + " fg:" + foreground);

        final long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mService) {
                //如果当前用户已经是需要切换的用户，退出当前流程
                final int oldUserId = mCurrentUserId;
                if (oldUserId == userId) {
                    return true;
                }

                mService.mStackSupervisor.setLockTaskModeLocked(null,
                        ActivityManager.LOCK_TASK_MODE_NONE, "startUser", false);
                final UserInfo userInfo = getUserInfo(userId);
                //如果没有需要启动的用户的信息，则直接退出
                if (userInfo == null) {
                    Slog.w(TAG, "No user info for user #" + userId);
                    return false;
                }
                //如果是前台启动且是profile用户，则直接退出
                if (foreground && userInfo.isManagedProfile()) {
                    Slog.w(TAG, "Cannot switch to User #" + userId + ": not a full user");
                    return false;
                }
                //如果是前台启动，则需要将屏幕冻结
                if (foreground) {
                    mService.mWindowManager.startFreezingScreen(
                            R.anim.screen_user_exit, R.anim.screen_user_enter);
                }

                boolean needStart = false;

                // If the user we are switching to is not currently started, then
                // we need to start it now.
                if (mStartedUsers.get(userId) == null) {
                    UserState userState = new UserState(UserHandle.of(userId));
                    mStartedUsers.put(userId, userState);
                    getUserManagerInternal().setUserState(userId, userState.state);
                    updateStartedUserArrayLocked();
                    needStart = true;
                }
                //调整用户在mUserLru列表中的位置，当前用户放在最后位置
                final UserState uss = mStartedUsers.get(userId);
                final Integer userIdInt = userId;
                mUserLru.remove(userIdInt);
                mUserLru.add(userIdInt);

                if (foreground) {
                    //如果是前台切换，直接切换到需要启动的用户
                    mCurrentUserId = userId;
                    mService.updateUserConfigurationLocked();
                    mTargetUserId = UserHandle.USER_NULL; // reset, mCurrentUserId has caught up
                   //更新与该用户相关的profile列表
                    updateCurrentProfileIdsLocked();
                    //在WMS中设置需要启动的用户为当前用户
                    mService.mWindowManager.setCurrentUser(userId, mCurrentProfileIds);
                    // Once the internal notion of the active user has switched, we lock the device
                    // with the option to show the user switcher on the keyguard.
                    mService.mWindowManager.lockNow(null);
                } else {
                    final Integer currentUserIdInt = mCurrentUserId;
                    //更新与该用户相关的profile列表
                    updateCurrentProfileIdsLocked();
                    mService.mWindowManager.setCurrentProfileIds(mCurrentProfileIds);
                    mUserLru.remove(currentUserIdInt);
                    mUserLru.add(currentUserIdInt);
                }

                // Make sure user is in the started state.  If it is currently
                // stopping, we need to knock that off.
                if (uss.state == UserState.STATE_STOPPING) {
                    // If we are stopping, we haven't sent ACTION_SHUTDOWN,
                    // so we can just fairly silently bring the user back from
                    // the almost-dead.
                    uss.setState(uss.lastState);
                    getUserManagerInternal().setUserState(userId, uss.state);
                    updateStartedUserArrayLocked();
                    needStart = true;
                } else if (uss.state == UserState.STATE_SHUTDOWN) {
                    // This means ACTION_SHUTDOWN has been sent, so we will
                    // need to treat this as a new boot of the user.
                    uss.setState(UserState.STATE_BOOTING);
                    getUserManagerInternal().setUserState(userId, uss.state);
                    updateStartedUserArrayLocked();
                    needStart = true;
                }

                if (uss.state == UserState.STATE_BOOTING) {
                    //如果用户的状态是正在启动，则发送一个SYSTEM_USER_START_MSG消息
                    // Give user manager a chance to propagate user restrictions
                    // to other services and prepare app storage
                    getUserManager().onBeforeStartUser(userId);

                    // Booting up a new user, need to tell system services about it.
                    // Note that this is on the same handler as scheduling of broadcasts,
                    // which is important because it needs to go first.
                    mHandler.sendMessage(mHandler.obtainMessage(SYSTEM_USER_START_MSG, userId, 0));
                }

                if (foreground) {
                    //发送SYSTEM_USER_CURRENT_MSG的消息
                    mHandler.sendMessage(mHandler.obtainMessage(SYSTEM_USER_CURRENT_MSG, userId,
                            oldUserId));
                    mHandler.removeMessages(REPORT_USER_SWITCH_MSG);
                    mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
                    //发送REPORT_USER_SWITCH_MSG和USER_SWITCH_TIMEOUT_MSG，防止切换时间过长
                    mHandler.sendMessage(mHandler.obtainMessage(REPORT_USER_SWITCH_MSG,
                            oldUserId, userId, uss));
                    mHandler.sendMessageDelayed(mHandler.obtainMessage(USER_SWITCH_TIMEOUT_MSG,
                            oldUserId, userId, uss), USER_SWITCH_TIMEOUT);
                }

                if (needStart) {
                    // Send USER_STARTED broadcast
                    Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                            | Intent.FLAG_RECEIVER_FOREGROUND);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                    mService.broadcastIntentLocked(null, null, intent,
                            null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                            null, false, false, MY_PID, SYSTEM_UID, userId);
                }

                if (foreground) {
                    //将user设置到前台
                    moveUserToForegroundLocked(uss, oldUserId, userId);
                } else {
                    mService.mUserController.finishUserBoot(uss);
                }

                if (needStart) {
                    Intent intent = new Intent(Intent.ACTION_USER_STARTING);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                    mService.broadcastIntentLocked(null, null, intent,
                            null, new IIntentReceiver.Stub() {
                                @Override
                                public void performReceive(Intent intent, int resultCode,
                                        String data, Bundle extras, boolean ordered, boolean sticky,
                                        int sendingUser) throws RemoteException {
                                }
                            }, 0, null, null,
                            new String[] {INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                            null, true, false, MY_PID, SYSTEM_UID, UserHandle.USER_ALL);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }

        return true;
    }
```
分析如下几个关键方法
moveUserToForegroundLocked:获取newUserId用户在切换之前的stack状态，以便将原来在前台的应用推到前台
```java
     void moveUserToForegroundLocked(UserState uss, int oldUserId, int newUserId) {
            boolean homeInFront = mService.mStackSupervisor.switchUserLocked(newUserId, uss);
            if (homeInFront) {
                //如果原来的stack是桌面，则启动桌面
                mService.startHomeActivityLocked(newUserId, "moveUserToForeground");
            } else {
                //如果是其他应用，则将该应用推到前台
                mService.mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            EventLogTags.writeAmSwitchUser(newUserId);
            sendUserSwitchBroadcastsLocked(oldUserId, newUserId);
        }
```

消息队列处理:在上述方法中我们看到发送很多handler消息，如下简单介绍一下这些消息的作用

SYSTEM_USER_START_MSG：通知BatteryStatsService用户切换的消息以及让SystemServiceManager通知各个SystemService调用startUser
```java
        case SYSTEM_USER_START_MSG: {
                mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
                        Integer.toString(msg.arg1), msg.arg1);
                mSystemServiceManager.startUser(msg.arg1);
                break;
            }
```
SYSTEM_USER_CURRENT_MSG:通知BatteryStatsService用户切换信息以及让SystemServiceManager通知各个SeysteService调用switchUser
```java
    case SYSTEM_USER_CURRENT_MSG: {
                    mBatteryStatsService.noteEvent(
                            BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_FINISH,
                            Integer.toString(msg.arg2), msg.arg2);
                    mBatteryStatsService.noteEvent(
                            BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
                            Integer.toString(msg.arg1), msg.arg1);
                    mSystemServiceManager.switchUser(msg.arg1);
                    break;
                }
```
REPORT_USER_SWITCH_MSG:如果系统中有对切换用户感兴趣的模块，可以调用AMS的registerUserSwitchObserver方法来注册观察对象。当有用户切换时AMS会通过调用
来通知这些模块，模块处理完成后调用参数中传递的callback来通知AMS。最后都调勇完成后会调用sendContinueUserSwitchLocked来继续进行切换的工作。
```java
     case REPORT_USER_SWITCH_MSG: {
         mUserController.dispatchUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
         break;
     }
     
         void dispatchUserSwitch(final UserState uss, final int oldUserId, final int newUserId) {
             Slog.d(TAG, "Dispatch onUserSwitching oldUser #" + oldUserId + " newUser #" + newUserId);
             final int observerCount = mUserSwitchObservers.beginBroadcast();
             if (observerCount > 0) {
                 final ArraySet<String> curWaitingUserSwitchCallbacks = new ArraySet<>();
                 synchronized (mService) {
                     uss.switching = true;
                     mCurWaitingUserSwitchCallbacks = curWaitingUserSwitchCallbacks;
                 }
                 final AtomicInteger waitingCallbacksCount = new AtomicInteger(observerCount);
                 for (int i = 0; i < observerCount; i++) {
                     try {
                         // Prepend with unique prefix to guarantee that keys are unique
                         final String name = "#" + i + " " + mUserSwitchObservers.getBroadcastCookie(i);
                         synchronized (mService) {
                             curWaitingUserSwitchCallbacks.add(name);
                         }
                         final IRemoteCallback callback = new IRemoteCallback.Stub() {
                             @Override
                             public void sendResult(Bundle data) throws RemoteException {
                                 synchronized (mService) {
                                     // Early return if this session is no longer valid
                                     if (curWaitingUserSwitchCallbacks
                                             != mCurWaitingUserSwitchCallbacks) {
                                         return;
                                     }
                                     curWaitingUserSwitchCallbacks.remove(name);
                                     // Continue switching if all callbacks have been notified
                                     if (waitingCallbacksCount.decrementAndGet() == 0) {
                                         sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
                                     }
                                 }
                             }
                         };
                         mUserSwitchObservers.getBroadcastItem(i).onUserSwitching(newUserId, callback);
                     } catch (RemoteException e) {
                     }
                 }
             } else {
                 synchronized (mService) {
                     sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
                 }
             }
             mUserSwitchObservers.finishBroadcast();
         }
         
```
sendContinueUserSwitchLocked会发出CONTINUE_USER_SWITCH_MSG消息
```java
        void sendContinueUserSwitchLocked(UserState uss, int oldUserId, int newUserId) {
            mCurWaitingUserSwitchCallbacks = null;
            mHandler.removeMessages(USER_SWITCH_TIMEOUT_MSG);
            mHandler.sendMessage(mHandler.obtainMessage(ActivityManagerService.CONTINUE_USER_SWITCH_MSG,
                    oldUserId, newUserId, uss));
        }
```
CONTINUE_USER_SWITCH_MSG事实上是调用continueUserSwitch方法
```java
        case CONTINUE_USER_SWITCH_MSG: {
            mUserController.continueUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
            break;
        }
```
continueUserSwitch将发送REPORT_USER_SWITCH_COMPLETE_MSG信息
```java
    void continueUserSwitch(UserState uss, int oldUserId, int newUserId) {
            Slog.d(TAG, "Continue user switch oldUser #" + oldUserId + ", newUser #" + newUserId);
            synchronized (mService) {
                mService.mWindowManager.stopFreezingScreen();
            }
            uss.switching = false;
            mHandler.removeMessages(REPORT_USER_SWITCH_COMPLETE_MSG);
            mHandler.sendMessage(mHandler.obtainMessage(REPORT_USER_SWITCH_COMPLETE_MSG,
                    newUserId, 0));
            stopGuestOrEphemeralUserIfBackground();
            stopBackgroundUsersIfEnforced(oldUserId);
        }
```
接收到该消息后调用dispatchUserSwitchComplete方法
```java
    case REPORT_USER_SWITCH_COMPLETE_MSG: {
                mUserController.dispatchUserSwitchComplete(msg.arg1);
            } break;
```
该方法将给注册观察onUserSwitchComplete方法回调
```java
     void dispatchUserSwitchComplete(int userId) {
            final int observerCount = mUserSwitchObservers.beginBroadcast();
            for (int i = 0; i < observerCount; i++) {
                try {
                    mUserSwitchObservers.getBroadcastItem(i).onUserSwitchComplete(userId);
                } catch (RemoteException e) {
                }
            }
            mUserSwitchObservers.finishBroadcast();
        }
```

USER_SWITCH_TIMEOUT_MSG:该消息的目的是为了防止用户切换时间太长，毕竟只有所有的观察者都处理完了才能继续进行切换用户的操作。该消息被handler处理后将调用timeoutUserSwitch。

```java
    case USER_SWITCH_TIMEOUT_MSG: {
                mUserController.timeoutUserSwitch((UserState) msg.obj, msg.arg1, msg.arg2);
                break;
            }
```
timeoutUserSwitch方法的本质是调用sendContinueUserSwitchLocked方法。该方法已描述。
```java
void timeoutUserSwitch(UserState uss, int oldUserId, int newUserId) {
        synchronized (mService) {
            Slog.wtf(TAG, "User switch timeout: from " + oldUserId + " to " + newUserId
                    + ". Observers that didn't send results: " + mCurWaitingUserSwitchCallbacks);
            sendContinueUserSwitchLocked(uss, oldUserId, newUserId);
        }
    }
```

上述操作我们发现新建的User正处于Booting的阶段，那么如何将User进入下个阶段呢?上述的方法有个finishUserBoot和moveUserToForegroundLocked方法。我们先看finishUserBoot

finishUserBoot将调用maybeUnlockUser。finishUserBoot将新创建的User从状态STATE_BOOTING设置为STATE_RUNNING_LOCKED
```java
    private void finishUserBoot(UserState uss, IIntentReceiver resultTo) {
        final int userId = uss.mHandle.getIdentifier();

        Slog.d(TAG, "Finishing user boot " + userId);
        synchronized (mService) {
            // Bail if we ended up with a stale user
            if (mStartedUsers.get(userId) != uss) return;

            // We always walk through all the user lifecycle states to send
            // consistent developer events. We step into RUNNING_LOCKED here,
            // but we might immediately step into RUNNING below if the user
            // storage is already unlocked.
            if (uss.setState(STATE_BOOTING, STATE_RUNNING_LOCKED)) {
                getUserManagerInternal().setUserState(userId, uss.state);

                int uptimeSeconds = (int)(SystemClock.elapsedRealtime() / 1000);
                MetricsLogger.histogram(mService.mContext, "framework_locked_boot_completed",
                    uptimeSeconds);

                Intent intent = new Intent(Intent.ACTION_LOCKED_BOOT_COMPLETED, null);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                intent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
                        | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
                mService.broadcastIntentLocked(null, null, intent, null, resultTo, 0, null, null,
                        new String[] { android.Manifest.permission.RECEIVE_BOOT_COMPLETED },
                        AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID, userId);
            }

            // We need to delay unlocking managed profiles until the parent user
            // is also unlocked.
            if (getUserManager().isManagedProfile(userId)) {
                final UserInfo parent = getUserManager().getProfileParent(userId);
                if (parent != null
                        && isUserRunningLocked(parent.id, ActivityManager.FLAG_AND_UNLOCKED)) {
                    Slog.d(TAG, "User " + userId + " (parent " + parent.id
                            + "): attempting unlock because parent is unlocked");
                    maybeUnlockUser(userId);
                } else {
                    String parentId = (parent == null) ? "<null>" : String.valueOf(parent.id);
                    Slog.d(TAG, "User " + userId + " (parent " + parentId
                            + "): delaying unlock because parent is locked");
                }
            } else {
                maybeUnlockUser(userId);
            }
        }
    }
```
maybeUnlockUser调用unlockUserCleared方法
```java
 boolean maybeUnlockUser(final int userId) {
        // Try unlocking storage using empty token
        return unlockUserCleared(userId, null, null, null);
    }
```
unlockUserCleared方法调用finishUserUnlocking方法
```java
boolean unlockUserCleared(final int userId, byte[] token, byte[] secret,
            IProgressListener listener) {
        UserState uss;
        synchronized (mService) {
            // TODO Move this block outside of synchronized if it causes lock contention
            if (!StorageManager.isUserKeyUnlocked(userId)) {
                final UserInfo userInfo = getUserInfo(userId);
                final IMountService mountService = getMountService();
                try {
                    // We always want to unlock user storage, even user is not started yet
                    mountService.unlockUserKey(userId, userInfo.serialNumber, token, secret);
                } catch (RemoteException | RuntimeException e) {
                    Slog.w(TAG, "Failed to unlock: " + e.getMessage());
                }
            }
            // Bail if user isn't actually running, otherwise register the given
            // listener to watch for unlock progress
            uss = mStartedUsers.get(userId);
            if (uss == null) {
                notifyFinished(userId, listener);
                return false;
            } else {
                uss.mUnlockProgress.addListener(listener);
                uss.tokenProvided = (token != null);
            }
        }

        finishUserUnlocking(uss);

        final ArraySet<Integer> childProfilesToUnlock = new ArraySet<>();
        synchronized (mService) {

            // We just unlocked a user, so let's now attempt to unlock any
            // managed profiles under that user.
            for (int i = 0; i < mStartedUsers.size(); i++) {
                final int testUserId = mStartedUsers.keyAt(i);
                final UserInfo parent = getUserManager().getProfileParent(testUserId);
                if (parent != null && parent.id == userId && testUserId != userId) {
                    Slog.d(TAG, "User " + testUserId + " (parent " + parent.id
                            + "): attempting unlock because parent was just unlocked");
                    childProfilesToUnlock.add(testUserId);
                }
            }
        }

        final int size = childProfilesToUnlock.size();
        for (int i = 0; i < size; i++) {
            maybeUnlockUser(childProfilesToUnlock.valueAt(i));
        }

        return true;
    }
```
finishUserUnlocking方法将新创建的User由STATE_RUNNING_LOCKED转为STATE_RUNNING_UNLOCKING。同时发送SYSTEM_USER_UNLOCK_MSG的消息
```java
private void finishUserUnlocking(final UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        boolean proceedWithUnlock = false;
        synchronized (mService) {
            // Bail if we ended up with a stale user
            if (mStartedUsers.get(uss.mHandle.getIdentifier()) != uss) return;

            // Only keep marching forward if user is actually unlocked
            if (!StorageManager.isUserKeyUnlocked(userId)) return;

            if (uss.setState(STATE_RUNNING_LOCKED, STATE_RUNNING_UNLOCKING)) {
                getUserManagerInternal().setUserState(userId, uss.state);
                proceedWithUnlock = true;
            }
        }

        if (proceedWithUnlock) {
            uss.mUnlockProgress.start();

            // Prepare app storage before we go any further
            uss.mUnlockProgress.setProgress(5,
                    mService.mContext.getString(R.string.android_start_title));
            mUserManager.onBeforeUnlockUser(userId);
            uss.mUnlockProgress.setProgress(20);

            // Dispatch unlocked to system services; when fully dispatched,
            // that calls through to the next "unlocked" phase
            mHandler.obtainMessage(SYSTEM_USER_UNLOCK_MSG, userId, 0, uss)
                    .sendToTarget();
        }
    }
```
SYSTEM_USER_UNLOCK_MSG消息实质上调用的是finishUserUnlocked方法
```java
     case SYSTEM_USER_UNLOCK_MSG: {
         final int userId = msg.arg1;
         mSystemServiceManager.unlockUser(userId);
         synchronized (ActivityManagerService.this) {
             mRecentTasks.loadUserRecentsLocked(userId);
         }
         if (userId == UserHandle.USER_SYSTEM) {
             startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_UNAWARE);
         }
         installEncryptionUnawareProviders(userId);
         mUserController.finishUserUnlocked((UserState) msg.obj);
         break;
     }
```
finishUserUnlocked将新创建的User由状态STATE_RUNNING_UNLOCKING设置为STATE_RUNNING_UNLOCKED。最后将调用finishUserUnlockedCompleted方法。
```java
void finishUserUnlocked(final UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        synchronized (mService) {
            // Bail if we ended up with a stale user
            if (mStartedUsers.get(uss.mHandle.getIdentifier()) != uss) return;

            // Only keep marching forward if user is actually unlocked
            if (!StorageManager.isUserKeyUnlocked(userId)) return;

            if (uss.setState(STATE_RUNNING_UNLOCKING, STATE_RUNNING_UNLOCKED)) {
                getUserManagerInternal().setUserState(userId, uss.state);
                uss.mUnlockProgress.finish();

                // Dispatch unlocked to external apps
                final Intent unlockedIntent = new Intent(Intent.ACTION_USER_UNLOCKED);
                unlockedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                unlockedIntent.addFlags(
                        Intent.FLAG_RECEIVER_REGISTERED_ONLY | Intent.FLAG_RECEIVER_FOREGROUND);
                mService.broadcastIntentLocked(null, null, unlockedIntent, null, null, 0, null,
                        null, null, AppOpsManager.OP_NONE, null, false, false, MY_PID, SYSTEM_UID,
                        userId);
                ......
                // Send PRE_BOOT broadcasts if user fingerprint changed; we
                // purposefully block sending BOOT_COMPLETED until after all
                // PRE_BOOT receivers are finished to avoid ANR'ing apps
                final UserInfo info = getUserInfo(userId);
                if (!Objects.equals(info.lastLoggedInFingerprint, Build.FINGERPRINT)) {
                    // Suppress double notifications for managed profiles that
                    // were unlocked automatically as part of their parent user
                    // being unlocked.
                    final boolean quiet;
                    if (info.isManagedProfile()) {
                        quiet = !uss.tokenProvided
                                || !mLockPatternUtils.isSeparateProfileChallengeEnabled(userId);
                    } else {
                        quiet = false;
                    }
                    new PreBootBroadcaster(mService, userId, null, quiet) {
                        @Override
                        public void onFinished() {
                            finishUserUnlockedCompleted(uss);
                        }
                    }.sendNext();
                } else {
                    finishUserUnlockedCompleted(uss);
                }
            }
        }
    }
```
在finishUserUnlockedCompleted中如果新建的User没有被initialized。会调用getUserManager().makeInitialized方法完成新建User的initialized。同时发出ACTION_BOOT_COMPLETED。这样就完成了新建User的启动。
```java
    private void finishUserUnlockedCompleted(UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        synchronized (mService) {
            // Bail if we ended up with a stale user
            if (mStartedUsers.get(uss.mHandle.getIdentifier()) != uss) return;
            final UserInfo userInfo = getUserInfo(userId);
            if (userInfo == null) {
                return;
            }

            // Only keep marching forward if user is actually unlocked
            if (!StorageManager.isUserKeyUnlocked(userId)) return;

            // Remember that we logged in
            mUserManager.onUserLoggedIn(userId);

            if (!userInfo.isInitialized()) {
                if (userId != UserHandle.USER_SYSTEM) {
                    Slog.d(TAG, "Initializing user #" + userId);
                    Intent intent = new Intent(Intent.ACTION_USER_INITIALIZE);
                    intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                    mService.broadcastIntentLocked(null, null, intent, null,
                            new IIntentReceiver.Stub() {
                                @Override
                                public void performReceive(Intent intent, int resultCode,
                                        String data, Bundle extras, boolean ordered,
                                        boolean sticky, int sendingUser) {
                                    // Note: performReceive is called with mService lock held
                                    getUserManager().makeInitialized(userInfo.id);
                                }
                            }, 0, null, null, null, AppOpsManager.OP_NONE,
                            null, true, false, MY_PID, SYSTEM_UID, userId);
                }
            }

            Slog.d(TAG, "Sending BOOT_COMPLETE user #" + userId);
            int uptimeSeconds = (int)(SystemClock.elapsedRealtime() / 1000);
            MetricsLogger.histogram(mService.mContext, "framework_boot_completed", uptimeSeconds);
            final Intent bootIntent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
            bootIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
            bootIntent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
                    | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
            mService.broadcastIntentLocked(null, null, bootIntent, null, null, 0, null, null,
                    new String[] { android.Manifest.permission.RECEIVE_BOOT_COMPLETED },
                    AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID, userId);
        }
    }
```
同时我们可以看到moveUserToForegroundLocked方法本质上启动Activity，我们知道Activity的启动流程一定会调用AMS中activityIdle方法，而activityIdle方法将调用mStackSupervisor.activityIdleInternalLocked方法。
而该方法中调用finishUserSwitch方法。
```java
    final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            Configuration config) {
       ......
        if (!booting) {
            // Complete user switch
            if (startingUsers != null) {
                for (int i = 0; i < startingUsers.size(); i++) {
                    mService.mUserController.finishUserSwitch(startingUsers.get(i));
                }
            }
        }

        mService.trimApplications();
        //dump();
        //mWindowManager.dump();

        if (activityRemoved) {
            resumeFocusedStackTopActivityLocked();
        }

        return r;
    }
```
finishUserSwitch方法将调用finishUserBoot方法。流程如上文所述。
```java
    void finishUserSwitch(UserState uss) {
        synchronized (mService) {
            finishUserBoot(uss);

            startProfilesLocked();
            stopRunningUsersLocked(MAX_RUNNING_USERS);
        }
    }
```
六.删除用户
-----------
删除用户的入口是UMS中的removeUser方法，该方法的实现如下:
```java
public boolean removeUser(int userHandle) {
        //权限检测
        checkManageOrCreateUsersPermission("Only the system can remove users");
        //检查一下该用户是否被限制删除用户
        if (getUserRestrictions(UserHandle.getCallingUserId()).getBoolean(
                UserManager.DISALLOW_REMOVE_USER, false)) {
            Log.w(LOG_TAG, "Cannot remove user. DISALLOW_REMOVE_USER is enabled.");
            return false;
        }

        long ident = Binder.clearCallingIdentity();
        try {
            final UserData userData;
            int currentUser = ActivityManager.getCurrentUser();
            if (currentUser == userHandle) {
                Log.w(LOG_TAG, "Current user cannot be removed");
                return false;
            }
            synchronized (mPackagesLock) {
                synchronized (mUsersLock) {
                    userData = mUsers.get(userHandle);
                     // 检查被删除的用户是不是管理员用户userHandle=0，检查用户列表中是否有该用户，以及该用户是否是正在被删除的用户
                    if (userHandle == 0 || userData == null || mRemovingUserIds.get(userHandle)) {
                        return false;
                    }

                    // We remember deleted user IDs to prevent them from being
                    // reused during the current boot; they can still be reused
                    // after a reboot.
                    mRemovingUserIds.put(userHandle, true);
                }

                try {
                    mAppOpsService.removeUser(userHandle);
                } catch (RemoteException e) {
                    Log.w(LOG_TAG, "Unable to notify AppOpsService of removing user", e);
                }
                // Set this to a partially created user, so that the user will be purged
                // on next startup, in case the runtime stops now before stopping and
                // removing the user completely.
                userData.info.partial = true;
                // Mark it as disabled, so that it isn't returned any more when
                // profiles are queried.
                userData.info.flags |= UserInfo.FLAG_DISABLED;
                //更新用户信息并写入到xml中
                writeUserLP(userData);
            }
            // 如果该user是一个user的一份profile，则发出一个ACTION_MANAGED_PROFILE_REMOVED广播
            if (userData.info.profileGroupId != UserInfo.NO_PROFILE_GROUP_ID
                    && userData.info.isManagedProfile()) {
                // Send broadcast to notify system that the user removed was a
                // managed user.
                sendProfileRemovedBroadcast(userData.info.profileGroupId, userData.info.id);
            }

            if (DBG) Slog.i(LOG_TAG, "Stopping user " + userHandle);
            int res;
            try {
                //调用AMS停止当前的用户
                res = ActivityManagerNative.getDefault().stopUser(userHandle, /* force= */ true,
                //设置回调函数，当stopUser执行完成之后调用finishRemoveUser方法
                new IStopUserCallback.Stub() {
                            @Override
                            public void userStopped(int userId) {
                                finishRemoveUser(userId);
                            }
                            @Override
                            public void userStopAborted(int userId) {
                            }
                        });
            } catch (RemoteException e) {
                return false;
            }
            return res == ActivityManager.USER_OP_SUCCESS;
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }

```
AMS中的stopUser方法,该方法将调用UserController中的stopUser方法
```java
@Override
    public int stopUser(final int userId, boolean force, final IStopUserCallback callback) {
        return mUserController.stopUser(userId, force, callback);
    }
```
UserController中的stopUser方法将调用stopUsersLocked方法
```java
    int stopUser(final int userId, final boolean force, final IStopUserCallback callback) {
        if (mService.checkCallingPermission(INTERACT_ACROSS_USERS_FULL)
                != PackageManager.PERMISSION_GRANTED) {
            String msg = "Permission Denial: switchUser() from pid="
                    + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid()
                    + " requires " + INTERACT_ACROSS_USERS_FULL;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
        if (userId < 0 || userId == UserHandle.USER_SYSTEM) {
            throw new IllegalArgumentException("Can't stop system user " + userId);
        }
        mService.enforceShellRestriction(UserManager.DISALLOW_DEBUGGING_FEATURES,
                userId);
        synchronized (mService) {
            return stopUsersLocked(userId, force, callback);
        }
    }
```
stopUsersLocked方法将调用stopUsersLocked
```java
    private int stopUsersLocked(final int userId, boolean force, final IStopUserCallback callback) {
        if (userId == UserHandle.USER_SYSTEM) {
            return USER_OP_ERROR_IS_SYSTEM;
        }
        if (isCurrentUserLocked(userId)) {
            return USER_OP_IS_CURRENT;
        }
        int[] usersToStop = getUsersToStopLocked(userId);
        // If one of related users is system or current, no related users should be stopped
        for (int i = 0; i < usersToStop.length; i++) {
            int relatedUserId = usersToStop[i];
            if ((UserHandle.USER_SYSTEM == relatedUserId) || isCurrentUserLocked(relatedUserId)) {
                if (DEBUG_MU) Slog.i(TAG, "stopUsersLocked cannot stop related user "
                        + relatedUserId);
                // We still need to stop the requested user if it's a force stop.
                if (force) {
                    Slog.i(TAG,
                            "Force stop user " + userId + ". Related users will not be stopped");
                    stopSingleUserLocked(userId, callback);
                    return USER_OP_SUCCESS;
                }
                return USER_OP_ERROR_RELATED_USERS_CANNOT_STOP;
            }
        }
        if (DEBUG_MU) Slog.i(TAG, "stopUsersLocked usersToStop=" + Arrays.toString(usersToStop));
        for (int userIdToStop : usersToStop) {
            stopSingleUserLocked(userIdToStop, userIdToStop == userId ? callback : null);
        }
        return USER_OP_SUCCESS;
    }
```
stopUsersLocked将调用stopSingleUserLocked。该方法会将User的状态设置为STATE_STOPPING。同时会调用了finishUserStopping
```java
    private void stopSingleUserLocked(final int userId, final IStopUserCallback callback) {
        if (DEBUG_MU) Slog.i(TAG, "stopSingleUserLocked userId=" + userId);
        final UserState uss = mStartedUsers.get(userId);
        if (uss == null) {
            // User is not started, nothing to do...  but we do need to
            // callback if requested.
            if (callback != null) {
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            callback.userStopped(userId);
                        } catch (RemoteException e) {
                        }
                    }
                });
            }
            return;
        }

        if (callback != null) {
            uss.mStopCallbacks.add(callback);
        }

        if (uss.state != UserState.STATE_STOPPING
                && uss.state != UserState.STATE_SHUTDOWN) {
            //将状态设置为正在停止
            uss.setState(UserState.STATE_STOPPING);
            getUserManagerInternal().setUserState(userId, uss.state);
            updateStartedUserArrayLocked();

            long ident = Binder.clearCallingIdentity();
            try {
                // We are going to broadcast ACTION_USER_STOPPING and then
                // once that is done send a final ACTION_SHUTDOWN and then
                // stop the user.
                final Intent stoppingIntent = new Intent(Intent.ACTION_USER_STOPPING);
                stoppingIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                stoppingIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                stoppingIntent.putExtra(Intent.EXTRA_SHUTDOWN_USERSPACE_ONLY, true);
                // This is the result receiver for the initial stopping broadcast.
                final IIntentReceiver stoppingReceiver = new IIntentReceiver.Stub() {
                    @Override
                    public void performReceive(Intent intent, int resultCode, String data,
                            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                        mHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                finishUserStopping(userId, uss);
                            }
                        });
                    }
                };
                // Clear broadcast queue for the user to avoid delivering stale broadcasts
                mService.clearBroadcastQueueForUserLocked(userId);
                // Kick things off.
                mService.broadcastIntentLocked(null, null, stoppingIntent,
                        null, stoppingReceiver, 0, null, null,
                        new String[]{INTERACT_ACROSS_USERS}, AppOpsManager.OP_NONE,
                        null, true, false, MY_PID, SYSTEM_UID, UserHandle.USER_ALL);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
    }
```
finishUserStopping方法将调用finishUserStopped方法并调用 mService.mSystemServiceManager.stopUser(userId)方法通知SystemService调用stopUser。
```java
    void finishUserStopping(final int userId, final UserState uss) {
        // On to the next.
        final Intent shutdownIntent = new Intent(Intent.ACTION_SHUTDOWN);
        // This is the result receiver for the final shutdown broadcast.
        final IIntentReceiver shutdownReceiver = new IIntentReceiver.Stub() {
            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        finishUserStopped(uss);
                    }
                });
            }
        };

        synchronized (mService) {
            if (uss.state != UserState.STATE_STOPPING) {
                // Whoops, we are being started back up.  Abort, abort!
                return;
            }
            uss.setState(UserState.STATE_SHUTDOWN);
        }
        getUserManagerInternal().setUserState(userId, uss.state);

        mService.mBatteryStatsService.noteEvent(
                BatteryStats.HistoryItem.EVENT_USER_RUNNING_FINISH,
                Integer.toString(userId), userId);
        mService.mSystemServiceManager.stopUser(userId);

        synchronized (mService) {
            mService.broadcastIntentLocked(null, null, shutdownIntent,
                    null, shutdownReceiver, 0, null, null, null,
                    AppOpsManager.OP_NONE,
                    null, true, false, MY_PID, SYSTEM_UID, userId);
        }
    }
```
finishUserStopped方法主要作用是将多用户中的app停止。
```java
void finishUserStopped(UserState uss) {
        final int userId = uss.mHandle.getIdentifier();
        boolean stopped;
        ArrayList<IStopUserCallback> callbacks;
        synchronized (mService) {
            callbacks = new ArrayList<>(uss.mStopCallbacks);
            if (mStartedUsers.get(userId) != uss) {
                stopped = false;
            } else if (uss.state != UserState.STATE_SHUTDOWN) {
                stopped = false;
            } else {
                stopped = true;
                // User can no longer run.
                mStartedUsers.remove(userId);
                getUserManagerInternal().removeUserState(userId);
                mUserLru.remove(Integer.valueOf(userId));
                updateStartedUserArrayLocked();

                mService.onUserStoppedLocked(userId);
                // Clean up all state and processes associated with the user.
                // Kill all the processes for the user.
                forceStopUserLocked(userId, "finish user");
            }
        }

        for (int i = 0; i < callbacks.size(); i++) {
            try {
                if (stopped) callbacks.get(i).userStopped(userId);
                else callbacks.get(i).userStopAborted(userId);
            } catch (RemoteException e) {
            }
        }

        if (stopped) {
            mService.mSystemServiceManager.cleanupUser(userId);
            synchronized (mService) {
                mService.mStackSupervisor.removeUserLocked(userId);
            }
            // Remove the user if it is ephemeral.
            if (getUserInfo(userId).isEphemeral()) {
                mUserManager.removeUser(userId);
            }
        }
```

如何AMS中StopUser完成了任务。该方法完成之后将执行finishRemoveUser方法.在该方法中执行removeUserState
```java
void finishRemoveUser(final int userHandle) {
        if (DBG) Slog.i(LOG_TAG, "finishRemoveUser " + userHandle);
        // Let other services shutdown any activity and clean up their state before completely
        // wiping the user's system directory and removing from the user list
        long ident = Binder.clearCallingIdentity();
        try {
            Intent addedIntent = new Intent(Intent.ACTION_USER_REMOVED);
            addedIntent.putExtra(Intent.EXTRA_USER_HANDLE, userHandle);
            mContext.sendOrderedBroadcastAsUser(addedIntent, UserHandle.ALL,
                    android.Manifest.permission.MANAGE_USERS,

                    new BroadcastReceiver() {
                        @Override
                        public void onReceive(Context context, Intent intent) {
                            if (DBG) {
                                Slog.i(LOG_TAG,
                                        "USER_REMOVED broadcast sent, cleaning up user data "
                                        + userHandle);
                            }
                            new Thread() {
                                @Override
                                public void run() {
                                    // Clean up any ActivityManager state
                                    LocalServices.getService(ActivityManagerInternal.class)
                                            .onUserRemoved(userHandle);
                                    removeUserState(userHandle);
                                }
                            }.start();
                        }
                    },

                    null, Activity.RESULT_OK, null, null);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```
该方法的主要作用是删除之前创建在/data/system/users/{userId}中配置的相关xml文件同时调用PMS中的方法删除相关app数据。
```java
    private void removeUserState(final int userHandle) {
        try {
            mContext.getSystemService(StorageManager.class).destroyUserKey(userHandle);
        } catch (IllegalStateException e) {
            // This may be simply because the user was partially created.
            Slog.i(LOG_TAG,
                "Destroying key for user " + userHandle + " failed, continuing anyway", e);
        }

        // Cleanup gatekeeper secure user id
        try {
            final IGateKeeperService gk = GateKeeper.getService();
            if (gk != null) {
                gk.clearSecureUserId(userHandle);
            }
        } catch (Exception ex) {
            Slog.w(LOG_TAG, "unable to clear GK secure user id");
        }

        // Cleanup package manager settings
        mPm.cleanUpUser(this, userHandle);

        // Clean up all data before removing metadata
        mPm.destroyUserData(userHandle,
                StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);

        // Remove this user from the list
        synchronized (mUsersLock) {
            mUsers.remove(userHandle);
            mIsUserManaged.delete(userHandle);
        }
        synchronized (mUserStates) {
            mUserStates.delete(userHandle);
        }
        synchronized (mRestrictionsLock) {
            mBaseUserRestrictions.remove(userHandle);
            mAppliedUserRestrictions.remove(userHandle);
            mCachedEffectiveUserRestrictions.remove(userHandle);
            mDevicePolicyLocalUserRestrictions.remove(userHandle);
        }
        // Update the user list
        synchronized (mPackagesLock) {
            writeUserListLP();
        }
        // Remove user file
        AtomicFile userFile = new AtomicFile(new File(mUsersDir, userHandle + XML_SUFFIX));
        userFile.delete();
        updateUserIds();
    }
```
参考资料:

1.http://www.heqiangfly.com/2017/03/06/android-source-code-analysis-usermanager/

2.http://gityuan.com/2016/11/20/user_manager/