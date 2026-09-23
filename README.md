function saveAuthenticatedAccount(
    config,
    user,
    authorization
) {

    config =
        ensureSavedAccountStructure(
            config
        );


    if (
        !user ||
        !user.sys_id
    ) {

        throw new Error(
            'Authenticated ServiceCall user is missing.'
        );
    }


    const accountKey =
        getSavedAccountKey(
            config.instanceUrl,
            user.sys_id
        );


    if (!accountKey) {

        throw new Error(
            'Unable to create the ServiceCall account key.'
        );
    }


    /*
     * Preserve anything already stored
     * for this account.
     */
    const existingAccount =
        config.savedAccounts[
            accountKey
        ] || {};


    const now =
        new Date().toISOString();


    config.savedAccounts[
        accountKey
    ] = {

        /*
         * Stable account identity.
         */
        accountKey:
            accountKey,

        instanceUrl:
            config.instanceUrl || '',

        userSysId:
            user.sys_id,

        name:
            user.name || '',

        userName:
            user.user_name || '',

        email:
            user.email || '',

        serviceCallId:
            user.servicecall_id || '',


        /*
         * Authorization snapshot.
         *
         * IMPORTANT:
         * This is for UI/account information.
         * We will ALWAYS re-check /me when
         * activating the account.
         */
        isServiceCallUser:
            authorization
                ?.is_servicecall_user ===
            true,

        isServiceCallAdmin:
            authorization
                ?.is_servicecall_admin ===
            true,


        /*
         * Each saved account owns its
         * own OAuth credentials.
         *
         * For the first migration these
         * values come from the existing
         * working single-account config.
         */
        accessToken:
            config.accessToken ||
            existingAccount.accessToken ||
            '',

        refreshToken:
            config.refreshToken ||
            existingAccount.refreshToken ||
            '',

        tokenType:
    config.tokenType ||
    existingAccount.tokenType ||
    'Bearer',

expiresIn:
    config.expiresIn ||
    existingAccount.expiresIn ||
    0,

tokenObtainedAt:
    config.tokenObtainedAt ||
    existingAccount.tokenObtainedAt ||
    0,

        /*
         * Useful later for ordering the
         * account chooser by recency.
         */
        addedAt:
            existingAccount.addedAt ||
            now,

        lastUsedAt:
            now
    };


    /*
     * This account becomes the currently
     * selected account.
     */
    config.activeAccountKey =
        accountKey;


    return {
        config:
            config,

        accountKey:
            accountKey,

        account:
            config.savedAccounts[
                accountKey
            ]
    };
}
