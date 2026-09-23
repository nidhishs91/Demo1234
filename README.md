async function checkNotificationsOnce() {

    try {

        /*
         * Only fetch unread notifications.
         *
         * We are NOT showing popups yet.
         * First we prove background detection works.
         */

        const result =
            await serviceCallApiRequest(
                '/notifications?page=1&page_size=20&filter=unread',
                'GET'
            );


        if (
            !result ||
            result.success !== true
        ) {

            console.warn(
                'ServiceCall notification check returned no valid result:',
                result
            );

            return;
        }


        /*
         * Support the notification array returned
         * by the ServiceCall notifications API.
         */

        const notifications =
            Array.isArray(result.notifications)
                ? result.notifications
                : [];


        if (
            notifications.length === 0
        ) {

            console.log(
                'ServiceCall notification check: no unread notifications.'
            );

            return;
        }


        console.log(
            'ServiceCall unread notifications detected:',
            notifications.length
        );


        /*
         * Detection only for now.
         *
         * DO NOT:
         * - show popup
         * - play sound
         * - mark as read
         *
         * We will add those after this test passes.
         */

        notifications.forEach(
            (notification) => {

                console.log(
                    'ServiceCall notification detected:',
                    {
                        sys_id:
                            notification.sys_id || '',

                        type:
                            notification.type || '',

                        title:
                            notification.title || '',

                        message:
                            notification.message || '',

                        meeting_sys_id:
                            notification.meeting_sys_id || ''
                    }
                );
            }
        );


    } catch (error) {

        console.error(
            'ServiceCall notification check failed:',
            error
        );


        /*
         * Authentication expiry is handled
         * consistently with our other monitors.
         */

        if (
            error.code ===
            'AUTHENTICATION_REQUIRED'
        ) {

            stopNotificationLoop();


            sendAuthStatus(
                'authentication_required',
                'Your ServiceCall authorization has expired. Please sign in again.'
            );
        }
    }
}
