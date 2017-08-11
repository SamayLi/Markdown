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