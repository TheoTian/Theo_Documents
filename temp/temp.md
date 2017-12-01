/packages/apps/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java

开始安装

```
private void startInstall() {
658        // Start subactivity to actually install the application
659        Intent newIntent = new Intent();
           ...
663        newIntent.setClass(this, InstallAppProgress.class);
           ...
684        startActivity(newIntent);
685        finish();
686    }
```
/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallAppProgress.java

```
private void doPackageStage(PackageManager pm, PackageInstaller.SessionParams params) {
269        final PackageInstaller packageInstaller = pm.getPackageInstaller();
270        PackageInstaller.Session session = null;
271        try {
272            final String packageLocation = mPackageURI.getPath();
273            final File file = new File(packageLocation);
274            final int sessionId = packageInstaller.createSession(params);
275            final byte[] buffer = new byte[65536];
276
277            session = packageInstaller.openSession(sessionId);
278
279            final InputStream in = new FileInputStream(file);
280            final long sizeBytes = file.length();
281            final OutputStream out = session.openWrite("PackageInstaller", 0, sizeBytes);
282            try {
283                int c;
284                while ((c = in.read(buffer)) != -1) {
285                    out.write(buffer, 0, c);
286                    if (sizeBytes > 0) {
287                        final float fraction = ((float) c / (float) sizeBytes);
288                        session.addProgress(fraction);
289                    }
290                }
291                session.fsync(out);
292            } finally {
293                IoUtils.closeQuietly(in);
294                IoUtils.closeQuietly(out);
295            }
296
297            // Create a PendingIntent and use it to generate the IntentSender
298            Intent broadcastIntent = new Intent(BROADCAST_ACTION);
299            PendingIntent pendingIntent = PendingIntent.getBroadcast(
300                    InstallAppProgress.this /*context*/,
301                    sessionId,
302                    broadcastIntent,
303                    PendingIntent.FLAG_UPDATE_CURRENT);
304            session.commit(pendingIntent.getIntentSender());
305        } catch (IOException e) {
306            onPackageInstalled(PackageInstaller.STATUS_FAILURE);
307        } finally {
308            IoUtils.closeQuietly(session);
309        }
310    }
```

打开SESSION

/frameworks/base/core/java/android/content/pm/PackageInstaller.java

```
  public @NonNull Session openSession(int sessionId) throws IOException {
316        try {
317            return new Session(mInstaller.openSession(sessionId));
               ...
324    }
```


/frameworks/base/services/core/java/com/android/server/pm/PackageInstallerService.java

```
752    private IPackageInstallerSession openSessionInternal(int sessionId) throws IOException {
753        synchronized (mSessions) {
754            final PackageInstallerSession session = mSessions.get(sessionId);
755            if (session == null || !isCallingUidOwner(session)) {
756                throw new SecurityException("Caller has no access to session " + sessionId);
757            }
758            session.open();
759            return session;
760        }
761    }
```

提交SESSION

/frameworks/base/core/java/android/content/pm/PackageInstaller.java

```
 public void commit(@NonNull IntentSender statusReceiver) {
814            try {
815                mSession.commit(statusReceiver);
816            } catch (RemoteException e) {
817                throw e.rethrowFromSystemServer();
818            }
819        }

```

/frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java

```
 @Override
486    public void commit(IntentSender statusReceiver) {
           ...
520        mHandler.obtainMessage(MSG_COMMIT, adapter.getBinder()).sendToTarget();
521    }

private void commitLocked() throws PackageManagerException {
524        if (mDestroyed) {
525            throw new PackageManagerException(INSTALL_FAILED_INTERNAL_ERROR, "Session destroyed");
526        }
527        if (!mSealed) {
528            throw new PackageManagerException(INSTALL_FAILED_INTERNAL_ERROR, "Session not sealed");
529        }
530
531        try {
532            resolveStageDir();
533        } catch (IOException e) {
534            throw new PackageManagerException(INSTALL_FAILED_CONTAINER_ERROR,
535                    "Failed to resolve stage location", e);
536        }
537
538        // Verify that stage looks sane with respect to existing application.
539        // This currently only ensures packageName, versionCode, and certificate
540        // consistency.
541        validateInstallLocked();
542
543        Preconditions.checkNotNull(mPackageName);
544        Preconditions.checkNotNull(mSignatures);
545        Preconditions.checkNotNull(mResolvedBaseFile);
546
547        if (!mPermissionsAccepted) {
548            // User needs to accept permissions; give installer an intent they
549            // can use to involve user.
550            final Intent intent = new Intent(PackageInstaller.ACTION_CONFIRM_PERMISSIONS);
551            intent.setPackage(mContext.getPackageManager().getPermissionControllerPackageName());
552            intent.putExtra(PackageInstaller.EXTRA_SESSION_ID, sessionId);
553            try {
554                mRemoteObserver.onUserActionRequired(intent);
555            } catch (RemoteException ignored) {
556            }
557
558            // Commit was keeping session marked as active until now; release
559            // that extra refcount so session appears idle.
560            close();
561            return;
562        }
563
564        if (stageCid != null) {
565            // Figure out the final installed size and resize the container once
566            // and for all. Internally the parser handles straddling between two
567            // locations when inheriting.
568            final long finalSize = calculateInstalledSize();
569            resizeContainer(stageCid, finalSize);
570        }
571
572        // Inherit any packages and native libraries from existing install that
573        // haven't been overridden.
574        if (params.mode == SessionParams.MODE_INHERIT_EXISTING) {
575            try {
576                final List<File> fromFiles = mResolvedInheritedFiles;
577                final File toDir = resolveStageDir();
578
579                if (LOGD) Slog.d(TAG, "Inherited files: " + mResolvedInheritedFiles);
580                if (!mResolvedInheritedFiles.isEmpty() && mInheritedFilesBase == null) {
581                    throw new IllegalStateException("mInheritedFilesBase == null");
582                }
583
584                if (isLinkPossible(fromFiles, toDir)) {
585                    if (!mResolvedInstructionSets.isEmpty()) {
586                        final File oatDir = new File(toDir, "oat");
587                        createOatDirs(mResolvedInstructionSets, oatDir);
588                    }
589                    linkFiles(fromFiles, toDir, mInheritedFilesBase);
590                } else {
591                    // TODO: this should delegate to DCS so the system process
592                    // avoids holding open FDs into containers.
593                    copyFiles(fromFiles, toDir);
594                }
595            } catch (IOException e) {
596                throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
597                        "Failed to inherit existing install", e);
598            }
599        }
600
601        // TODO: surface more granular state from dexopt
602        mInternalProgress = 0.5f;
603        computeProgressLocked(true);
604
605        // Unpack native libraries
606        extractNativeLibraries(mResolvedStageDir, params.abiOverride);
607
608        // Container is ready to go, let's seal it up!
609        if (stageCid != null) {
610            finalizeAndFixContainer(stageCid);
611        }
612
613        // We've reached point of no return; call into PMS to install the stage.
614        // Regardless of success or failure we always destroy session.
615        final IPackageInstallObserver2 localObserver = new IPackageInstallObserver2.Stub() {
616            @Override
617            public void onUserActionRequired(Intent intent) {
618                throw new IllegalStateException();
619            }
620
621            @Override
622            public void onPackageInstalled(String basePackageName, int returnCode, String msg,
623                    Bundle extras) {
624                destroyInternal();
625                dispatchSessionFinished(returnCode, msg, extras);
626            }
627        };
628
629        final UserHandle user;
630        if ((params.installFlags & PackageManager.INSTALL_ALL_USERS) != 0) {
631            user = UserHandle.ALL;
632        } else {
633            user = new UserHandle(userId);
634        }
635
636        mRelinquished = true;
637        mPm.installStage(mPackageName, stageDir, stageCid, localObserver, params,
638                installerPackageName, installerUid, user, mCertificates);
639    }
```

/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```
11549    void installStage(String packageName, File stagedDir, String stagedCid,
11550            IPackageInstallObserver2 observer, PackageInstaller.SessionParams sessionParams,
11551            String installerPackageName, int installerUid, UserHandle user,
11552            Certificate[][] certificates) {
11553        if (DEBUG_EPHEMERAL) {
11554            if ((sessionParams.installFlags & PackageManager.INSTALL_EPHEMERAL) != 0) {
11555                Slog.d(TAG, "Ephemeral install of " + packageName);
11556            }
11557        }
11558        final VerificationInfo verificationInfo = new VerificationInfo(
11559                sessionParams.originatingUri, sessionParams.referrerUri,
11560                sessionParams.originatingUid, installerUid);
11561
11562        final OriginInfo origin;
11563        if (stagedDir != null) {
11564            origin = OriginInfo.fromStagedFile(stagedDir);
11565        } else {
11566            origin = OriginInfo.fromStagedContainer(stagedCid);
11567        }
11568
11569        final Message msg = mHandler.obtainMessage(INIT_COPY);
11570        final InstallParams params = new InstallParams(origin, null, observer,
11571                sessionParams.installFlags, installerPackageName, sessionParams.volumeUuid,
11572                verificationInfo, user, sessionParams.abiOverride,
11573                sessionParams.grantedRuntimePermissions, certificates);
11574        params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
11575        msg.obj = params;
11576
11577        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
11578                System.identityHashCode(msg.obj));
11579        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
11580                System.identityHashCode(msg.obj));
11581
11582        mHandler.sendMessage(msg);
11583    }
11584
    
/*
         * Invoke remote method to get package information and install
         * location values. Override install location based on default
         * policy if needed and then create install arguments based
         * on the install location.
         */
        public void handleStartCopy() throws RemoteException {
            int ret = PackageManager.INSTALL_SUCCEEDED;

            // If we're already staged, we've firmly committed to an install location
            if (origin.staged) {
                if (origin.file != null) {
                    installFlags |= PackageManager.INSTALL_INTERNAL;
                    installFlags &= ~PackageManager.INSTALL_EXTERNAL;
                } else if (origin.cid != null) {
                    installFlags |= PackageManager.INSTALL_EXTERNAL;
                    installFlags &= ~PackageManager.INSTALL_INTERNAL;
                } else {
                    throw new IllegalStateException("Invalid stage location");
                }
            }

            final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
            final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
            final boolean ephemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
            PackageInfoLite pkgLite = null;

            if (onInt && onSd) {
                // Check if both bits are set.
                Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else if (onSd && ephemeral) {
                Slog.w(TAG,  "Conflicting flags specified for installing ephemeral on external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else {
                pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                        packageAbiOverride);

                if (DEBUG_EPHEMERAL && ephemeral) {
                    Slog.v(TAG, "pkgLite for install: " + pkgLite);
                }

                /*
                 * If we have too little free space, try to free cache
                 * before giving up.
                 */
                if (!origin.staged && pkgLite.recommendedInstallLocation
                        == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                    // TODO: focus freeing disk space on the target device
                    final StorageManager storage = StorageManager.from(mContext);
                    final long lowThreshold = storage.getStorageLowBytes(
                            Environment.getDataDirectory());

                    final long sizeBytes = mContainerService.calculateInstalledSize(
                            origin.resolvedPath, isForwardLocked(), packageAbiOverride);

                    try {
                        mInstaller.freeCache(null, sizeBytes + lowThreshold, 0, 0);
                        pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath,
                                installFlags, packageAbiOverride);
                    } catch (InstallerException e) {
                        Slog.w(TAG, "Failed to free cache", e);
                    }

                    /*
                     * The cache free must have deleted the file we
                     * downloaded to install.
                     *
                     * TODO: fix the "freeCache" call to not delete
                     *       the file we care about.
                     */
                    if (pkgLite.recommendedInstallLocation
                            == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                        pkgLite.recommendedInstallLocation
                            = PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;
                    }
                }
            }

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                int loc = pkgLite.recommendedInstallLocation;
                if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
                    ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                    ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_APK;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_URI;
                } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
                    ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;
                } else {
                    // Override with defaults if needed.
                    loc = installLocationPolicy(pkgLite);
                    if (loc == PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE) {
                        ret = PackageManager.INSTALL_FAILED_VERSION_DOWNGRADE;
                    } else if (!onSd && !onInt) {
                        // Override install location with flags
                        if (loc == PackageHelper.RECOMMEND_INSTALL_EXTERNAL) {
                            // Set the flag to install on external media.
                            installFlags |= PackageManager.INSTALL_EXTERNAL;
                            installFlags &= ~PackageManager.INSTALL_INTERNAL;
                        } else if (loc == PackageHelper.RECOMMEND_INSTALL_EPHEMERAL) {
                            if (DEBUG_EPHEMERAL) {
                                Slog.v(TAG, "...setting INSTALL_EPHEMERAL install flag");
                            }
                            installFlags |= PackageManager.INSTALL_INSTANT_APP;
                            installFlags &= ~(PackageManager.INSTALL_EXTERNAL
                                    |PackageManager.INSTALL_INTERNAL);
                        } else {
                            // Make sure the flag for installing on external
                            // media is unset
                            installFlags |= PackageManager.INSTALL_INTERNAL;
                            installFlags &= ~PackageManager.INSTALL_EXTERNAL;
                        }
                    }
                }
            }

            final InstallArgs args = createInstallArgs(this);
            mArgs = args;

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                // TODO: http://b/22976637
                // Apps installed for "all" users use the device owner to verify the app
                UserHandle verifierUser = getUser();
                if (verifierUser == UserHandle.ALL) {
                    verifierUser = UserHandle.SYSTEM;
                }

                /*
                 * Determine if we have any installed package verifiers. If we
                 * do, then we'll defer to them to verify the packages.
                 */
                final int requiredUid = mRequiredVerifierPackage == null ? -1
                        : getPackageUid(mRequiredVerifierPackage, MATCH_DEBUG_TRIAGED_MISSING,
                                verifierUser.getIdentifier());
                final int installerUid =
                        verificationInfo == null ? -1 : verificationInfo.installerUid;
                if (!origin.existing && requiredUid != -1
                        && isVerificationEnabled(
                                verifierUser.getIdentifier(), installFlags, installerUid)) {
                    final Intent verification = new Intent(
                            Intent.ACTION_PACKAGE_NEEDS_VERIFICATION);
                    verification.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                    verification.setDataAndType(Uri.fromFile(new File(origin.resolvedPath)),
                            PACKAGE_MIME_TYPE);
                    verification.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);

                    // Query all live verifiers based on current user state
                    final List<ResolveInfo> receivers = queryIntentReceiversInternal(verification,
                            PACKAGE_MIME_TYPE, 0, verifierUser.getIdentifier());

                    if (DEBUG_VERIFY) {
                        Slog.d(TAG, "Found " + receivers.size() + " verifiers for intent "
                                + verification.toString() + " with " + pkgLite.verifiers.length
                                + " optional verifiers");
                    }

                    final int verificationId = mPendingVerificationToken++;

                    verification.putExtra(PackageManager.EXTRA_VERIFICATION_ID, verificationId);

                    verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALLER_PACKAGE,
                            installerPackageName);

                    verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALL_FLAGS,
                            installFlags);

                    verification.putExtra(PackageManager.EXTRA_VERIFICATION_PACKAGE_NAME,
                            pkgLite.packageName);

                    verification.putExtra(PackageManager.EXTRA_VERIFICATION_VERSION_CODE,
                            pkgLite.versionCode);

                    if (verificationInfo != null) {
                        if (verificationInfo.originatingUri != null) {
                            verification.putExtra(Intent.EXTRA_ORIGINATING_URI,
                                    verificationInfo.originatingUri);
                        }
                        if (verificationInfo.referrer != null) {
                            verification.putExtra(Intent.EXTRA_REFERRER,
                                    verificationInfo.referrer);
                        }
                        if (verificationInfo.originatingUid >= 0) {
                            verification.putExtra(Intent.EXTRA_ORIGINATING_UID,
                                    verificationInfo.originatingUid);
                        }
                        if (verificationInfo.installerUid >= 0) {
                            verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALLER_UID,
                                    verificationInfo.installerUid);
                        }
                    }

                    final PackageVerificationState verificationState = new PackageVerificationState(
                            requiredUid, args);

                    mPendingVerification.append(verificationId, verificationState);

                    final List<ComponentName> sufficientVerifiers = matchVerifiers(pkgLite,
                            receivers, verificationState);

                    DeviceIdleController.LocalService idleController = getDeviceIdleController();
                    final long idleDuration = getVerificationTimeout();

                    /*
                     * If any sufficient verifiers were listed in the package
                     * manifest, attempt to ask them.
                     */
                    if (sufficientVerifiers != null) {
                        final int N = sufficientVerifiers.size();
                        if (N == 0) {
                            Slog.i(TAG, "Additional verifiers required, but none installed.");
                            ret = PackageManager.INSTALL_FAILED_VERIFICATION_FAILURE;
                        } else {
                            for (int i = 0; i < N; i++) {
                                final ComponentName verifierComponent = sufficientVerifiers.get(i);
                                idleController.addPowerSaveTempWhitelistApp(Process.myUid(),
                                        verifierComponent.getPackageName(), idleDuration,
                                        verifierUser.getIdentifier(), false, "package verifier");

                                final Intent sufficientIntent = new Intent(verification);
                                sufficientIntent.setComponent(verifierComponent);
                                mContext.sendBroadcastAsUser(sufficientIntent, verifierUser);
                            }
                        }
                    }

                    final ComponentName requiredVerifierComponent = matchComponentForVerifier(
                            mRequiredVerifierPackage, receivers);
                    if (ret == PackageManager.INSTALL_SUCCEEDED
                            && mRequiredVerifierPackage != null) {
                        Trace.asyncTraceBegin(
                                TRACE_TAG_PACKAGE_MANAGER, "verification", verificationId);
                        /*
                         * Send the intent to the required verification agent,
                         * but only start the verification timeout after the
                         * target BroadcastReceivers have run.
                         */
                        verification.setComponent(requiredVerifierComponent);
                        idleController.addPowerSaveTempWhitelistApp(Process.myUid(),
                                mRequiredVerifierPackage, idleDuration,
                                verifierUser.getIdentifier(), false, "package verifier");
                        mContext.sendOrderedBroadcastAsUser(verification, verifierUser,
                                android.Manifest.permission.PACKAGE_VERIFICATION_AGENT,
                                new BroadcastReceiver() {
                                    @Override
                                    public void onReceive(Context context, Intent intent) {
                                        final Message msg = mHandler
                                                .obtainMessage(CHECK_PENDING_VERIFICATION);
                                        msg.arg1 = verificationId;
                                        mHandler.sendMessageDelayed(msg, getVerificationTimeout());
                                    }
                                }, null, 0, null, null);

                        /*
                         * We don't want the copy to proceed until verification
                         * succeeds, so null out this field.
                         */
                        mArgs = null;
                    }
                } else {
                    /*
                     * No package verification is enabled, so immediately start
                     * the remote call to initiate copy using temporary file.
                     */
                    ret = args.copyApk(mContainerService, true);
                }
            }

            mRet = ret;
        }
```


