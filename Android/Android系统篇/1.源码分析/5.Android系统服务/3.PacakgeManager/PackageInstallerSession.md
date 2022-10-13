我们来看out对象的来源，它通过 Session 的 openWrite 方法获得：

```java
        public @NonNull OutputStream openWrite(@NonNull String name, long offsetBytes,
                long lengthBytes) throws IOException {
            try {
                if (ENABLE_REVOCABLE_FD) {
                    return new ParcelFileDescriptor.AutoCloseOutputStream(
                            mSession.openWrite(name, offsetBytes, lengthBytes));
                } else {
                    final ParcelFileDescriptor clientSocket = mSession.openWrite(name,
                            offsetBytes, lengthBytes);
                    return new FileBridge.FileBridgeOutputStream(clientSocket);
                }
            } catch (RuntimeException e) {
                ExceptionUtils.maybeUnwrapIOException(e);
                throw e;
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
```

返回的 OutputStram 根据标志位 ENABLE_REVOCABLE_FD 来决定是  ParcelFileDescriptor.AutoCloseOutputStream 还是 FileBridge.FileBridgeOutputStream。这两者的区别暂不深究，我们可以看到他们都需要传入一个 ParcelFileDescriptor 对象作为参数。ParcelFileDescriptor  通过 mSession 的 openWrite 返回：

```java
    @Override
    public ParcelFileDescriptor openWrite(String name, long offsetBytes, long lengthBytes) {
        assertCanWrite(false);
        try {
            return doWriteInternal(name, offsetBytes, lengthBytes, null);
        } catch (IOException e) {
            throw ExceptionUtils.wrap(e);
        }
    }
    
    
    private ParcelFileDescriptor doWriteInternal(String name, long offsetBytes, long lengthBytes,
            ParcelFileDescriptor incomingFd) throws IOException {
		//......
        try {
            final File target;
            final long identity = Binder.clearCallingIdentity();
            try {
                target = new File(stageDir, name);
            } finally {
                Binder.restoreCallingIdentity(identity);
            }
            //......
        } catch (ErrnoException e) {
            throw e.rethrowAsIOException();
        }
    }

```

可以看到，目标文件 target 的路径是 “ stageDir / name ” 那么是在上面的 InstallingAsyncTask#doInBackground中传入的 “PackageInstaller”，而 stageDir 是在 InstallInstalling 的 onCreate 方法中调用 `                    mSessionId = getPackageManager().getPackageInstaller().createSession(params);` 设置的。这个过程中会跨进程调用 PackageInstallerService 的 createSessionInternal 方法

```java
        File stageDir = null;
        String stageCid = null;
        if (!params.isMultiPackage) {
            if ((params.installFlags & PackageManager.INSTALL_INTERNAL) != 0) {
                stageDir = buildSessionDir(sessionId, params);
            } else {
                stageCid = buildExternalStageCid(sessionId);
            }
        }
        
 private File buildSessionDir(int sessionId, SessionParams params) {
        if (params.isStaged || (params.installFlags & PackageManager.INSTALL_APEX) != 0) {
            final File sessionStagingDir = Environment.getDataStagingDirectory(params.volumeUuid);
            return new File(sessionStagingDir, "session_" + sessionId);
        }
        return buildTmpSessionDir(sessionId, params.volumeUuid);
    }

    private File getTmpSessionDir(String volumeUuid) {
        return Environment.getDataAppDirectory(volumeUuid);
    }

    private File buildTmpSessionDir(int sessionId, String volumeUuid) {
        final File sessionStagingDir = getTmpSessionDir(volumeUuid);
        return new File(sessionStagingDir, "vmdl" + sessionId + ".tmp");
    }

    private String buildExternalStageCid(int sessionId) {
        return "smdl" + sessionId + ".tmp";
    }
```



```java
public static File getDataDirectory(String volumeUuid) {
    if (TextUtils.isEmpty(volumeUuid)) {
        return DIR_ANDROID_DATA;
    } else {
        return new File("/mnt/expand/" + volumeUuid);
    }
}
```



即 stageDir 可能为空，或者是 /data 目录 或者是 /mnt/expand/ volumeUuid目录



doInBackground 看完接着看 onPostExecute;

```java
        @Override
        protected void onPostExecute(PackageInstaller.Session session) {
            if (session != null) {
                Intent broadcastIntent = new Intent(BROADCAST_ACTION);
                broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                broadcastIntent.setPackage(getPackageName());
                broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);

                PendingIntent pendingIntent = PendingIntent.getBroadcast(
                        InstallInstalling.this,
                        mInstallId,
                        broadcastIntent,
                        PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_MUTABLE);

                session.commit(pendingIntent.getIntentSender());
                mCancelButton.setEnabled(false);
                setFinishOnTouchOutside(false);
            } else {
                getPackageManager().getPackageInstaller().abandonSession(mSessionId);

                if (!isCancelled()) {
                    launchFailure(PackageInstaller.STATUS_FAILURE,
                            PackageManager.INSTALL_FAILED_INVALID_APK, null);
                }
            }
        }
```

onPostExecute 回调中，如果传输成功，则 Session 对象不为null，接着会创建了一个 PendingIntent 对象，并通过 Session 的 commit 方法将 PendingIntent 的 IntentSender 传递出去。 其他进程可以通过 IntentSender 来发送 PendingIntent 中的 Intent。那么 IntentSender 被发送到哪里去了？我们可以看 Session 的 commit 方法:

```java
        public void commit(@NonNull IntentSender statusReceiver) {
            try {
                mSession.commit(statusReceiver, false);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
```

最终调用 PackageInstallerSession 的commit方法。

```java
    @Override
    public void commit(@NonNull IntentSender statusReceiver, boolean forTransfer) {
        if (hasParentSessionId()) {
            throw new IllegalStateException(
                    "Session " + sessionId + " is a child of multi-package session "
                            + getParentSessionId() +  " and may not be committed directly.");
        }

        if (!markAsSealed(statusReceiver, forTransfer)) {
            return;
        }
        if (isMultiPackage()) {
            synchronized (mLock) {
                final IntentSender childIntentSender =
                        new ChildStatusIntentReceiver(mChildSessions.clone(), statusReceiver)
                                .getIntentSender();
                boolean sealFailed = false;
                for (int i = mChildSessions.size() - 1; i >= 0; --i) {
                    if (!mChildSessions.valueAt(i)
                            .markAsSealed(childIntentSender, forTransfer)) {
                        sealFailed = true;
                    }
                }
                if (sealFailed) {
                    return;
                }
            }
        }

        dispatchSessionSealed();
    }

```

密封包，多个包就多次调用 markAsSealed方法，单个就调用一次。最后调用dispatchSessionSealed。

```java
    private void dispatchSessionSealed() {
        mHandler.obtainMessage(MSG_ON_SESSION_SEALED).sendToTarget();
    }
```

dispatchSessionSealed 通过 mHandler发送消息，mHandler 定义在 PackageInstallerSession中。

```java
    private final Handler.Callback mHandlerCallback = new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_ON_SESSION_SEALED:
                    handleSessionSealed();
                    break;
                case MSG_STREAM_VALIDATE_AND_COMMIT:
                    handleStreamValidateAndCommit();
                    break;
                case MSG_INSTALL:
                    handleInstall();
                    break;
                case MSG_ON_PACKAGE_INSTALLED:
                    final SomeArgs args = (SomeArgs) msg.obj;
                    final String packageName = (String) args.arg1;
                    final String message = (String) args.arg2;
                    final Bundle extras = (Bundle) args.arg3;
                    final IntentSender statusReceiver = (IntentSender) args.arg4;
                    final int returnCode = args.argi1;
                    args.recycle();

                    sendOnPackageInstalled(mContext, statusReceiver, sessionId,
                            isInstallerDeviceOwnerOrAffiliatedProfileOwner(), userId,
                            packageName, returnCode, message, extras);

                    break;
                case MSG_SESSION_VALIDATION_FAILURE:
                    final int error = msg.arg1;
                    final String detailMessage = (String) msg.obj;
                    onSessionValidationFailure(error, detailMessage);
                    break;
            }

            return true;
        }
    };

```



