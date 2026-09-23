(function process(request, response) {

    try {

        /* -----------------------------------------
           AUTHENTICATED USER
        ----------------------------------------- */

        var currentUserSysId =
            String(gs.getUserID() || '');

        if (!currentUserSysId) {

            response.setStatus(401);

            return {
                success: false,
                code: 'AUTHENTICATION_REQUIRED',
                message: 'Authentication is required.'
            };
        }


        /* -----------------------------------------
           FIND SERVICECALL USER
        ----------------------------------------- */

        var serviceCallUserGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_user'
            );

        serviceCallUserGR.addQuery(
            'u_user',
            currentUserSysId
        );

        serviceCallUserGR.setLimit(1);
        serviceCallUserGR.query();


        /*
         * If the authenticated user does not yet
         * have a ServiceCall User record, they
         * simply have no notifications.
         */
        if (!serviceCallUserGR.next()) {

            response.setStatus(200);

            return {
                success: true,
                notifications: [],
                unread_count: 0,
                page: 1,
                page_size: 20,
                has_more: false
            };
        }


        var serviceCallUserSysId =
            String(
                serviceCallUserGR.getUniqueValue()
            );


        /* -----------------------------------------
           QUERY PARAMETERS / PAGINATION
        ----------------------------------------- */

        var queryParams =
            request.queryParams || {};


        var page =
            parseInt(
                queryParams.page,
                10
            ) || 1;


        var pageSize =
            parseInt(
                queryParams.page_size,
                10
            ) || 20;

        /*
         * -----------------------------------------
         * SEARCH
         * -----------------------------------------
         *
         * Search is always applied AFTER recipient
         * security, so users can only search their
         * own ServiceCall notifications.
         */

        var search =
            String(
                request.queryParams.search ||
                ''
            ).trim();


        /*
         * Keep the search reasonably sized.
         */
        if (search.length > 100) {

            search =
                search.substring(
                    0,
                    100
                );
        }


        if (page < 1) {
            page = 1;
        }


        if (pageSize < 1) {
            pageSize = 20;
        }


        /*
         * Protect the API from unnecessarily
         * large requests.
         */
        if (pageSize > 50) {
            pageSize = 50;
        }


        var offset =
            (page - 1) * pageSize;


        /* -----------------------------------------
   UNREAD COUNT
----------------------------------------- */
        var unreadGA =
            new GlideAggregate(
                'x_1806573_servic_0_servicecall_notification'
            );

        unreadGA.addQuery(
            'u_recipient',
            serviceCallUserSysId
        );

        unreadGA.addQuery(
            'u_read',
            false
        );

        unreadGA.addAggregate(
            'COUNT'
        );

        unreadGA.query();

        var unreadCount = 0;

        if (unreadGA.next()) {

            unreadCount =
                parseInt(
                    unreadGA.getAggregate(
                        'COUNT'
                    ),
                    10
                ) || 0;
        }

        /* -----------------------------------------
           LOAD NOTIFICATIONS
        ----------------------------------------- */

        var notificationGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_notification'
            );

        notificationGR.addQuery(
            'u_recipient',
            serviceCallUserSysId
        );

        /*
         * -----------------------------------------
         * OPTIONAL SEARCH
         * -----------------------------------------
         *
         * Search:
         *
         * Number
         * Title
         * Message
         */

        if (search) {

            var searchQuery =
                notificationGR.addQuery(
                    'number',
                    'CONTAINS',
                    search
                );


            searchQuery.addOrCondition(
                'u_title',
                'CONTAINS',
                search
            );


            searchQuery.addOrCondition(
                'u_message',
                'CONTAINS',
                search
            );
        }


        /*
         * Newest notification first.
         */
        notificationGR.orderByDesc(
            'sys_created_on'
        );


        /*
         * Load one extra record.
         *
         * Example:
         * page size = 20
         *
         * Request 21 records.
         *
         * If record 21 exists, has_more = true.
         */
        notificationGR.chooseWindow(
            offset,
            offset + pageSize + 1
        );

        notificationGR.query();


        var notifications = [];
        var hasMore = false;


        while (notificationGR.next()) {

            /*
             * The extra record proves another
             * page exists. Do not include it
             * in this page's response.
             */
            if (
                notifications.length >=
                pageSize
            ) {

                hasMore = true;
                break;
            }


            var meetingSysId =
                String(
                    notificationGR.getValue(
                        'u_meeting'
                    ) || ''
                );


            var callSysId =
                String(
                    notificationGR.getValue(
                        'u_call'
                    ) || ''
                );


            var readValue =
                String(
                    notificationGR.getValue(
                        'u_read'
                    ) || ''
                );

            notifications.push({

                sys_id: String(
                    notificationGR.getUniqueValue()
                ),


                number: String(
                    notificationGR.getDisplayValue(
                        'number'
                    ) || ''
                ),


                type: String(
                    notificationGR.getValue(
                        'u_type'
                    ) || ''
                ),


                type_display: String(
                    notificationGR.getDisplayValue(
                        'u_type'
                    ) || ''
                ),


                title: String(
                    notificationGR.getValue(
                        'u_title'
                    ) || ''
                ),


                message: String(
                    notificationGR.getValue(
                        'u_message'
                    ) || ''
                ),


                read: (
                    readValue === '1' ||
                    readValue === 'true'
                ),


                read_at: String(
                    notificationGR.getValue(
                        'u_read_at'
                    ) || ''
                ),


                action_type: String(
                    notificationGR.getValue(
                        'u_action_type'
                    ) || ''
                ),


                action_type_display: String(
                    notificationGR.getDisplayValue(
                        'u_action_type'
                    ) || ''
                ),


                priority: String(
                    notificationGR.getValue(
                        'u_priority'
                    ) || ''
                ),


                priority_display: String(
                    notificationGR.getDisplayValue(
                        'u_priority'
                    ) || ''
                ),


                meeting_sys_id: meetingSysId,


                meeting_number: meetingSysId ?
                    String(
                        notificationGR.getDisplayValue(
                            'u_meeting'
                        ) || ''
                    ) : '',


                call_sys_id: callSysId,


                call_number: callSysId ?
                    String(
                        notificationGR.getDisplayValue(
                            'u_call'
                        ) || ''
                    ) : '',


                created_at: String(
                    notificationGR.getValue(
                        'sys_created_on'
                    ) || ''
                ),


                created_at_display: String(
                    notificationGR.getDisplayValue(
                        'sys_created_on'
                    ) || ''
                )
            });
        }


        /* -----------------------------------------
           UPDATE SERVICECALL LAST ACTIVE
        ----------------------------------------- */

        serviceCallUserGR.setValue(
            'u_last_active_at',
            new GlideDateTime()
        );

        serviceCallUserGR.update();


        /* -----------------------------------------
           RESPONSE
        ----------------------------------------- */

        response.setStatus(200);


        return {

            success: true,

            notifications: notifications,

            unread_count: unreadCount,

            page: page,

            page_size: pageSize,

            has_more: hasMore
        };


    } catch (error) {

        gs.error(
            'ServiceCall Get Notifications failed: ' +
            error.message
        );


        response.setStatus(500);


        return {
            success: false,
            code: 'NOTIFICATIONS_FAILED',
            message: 'Unable to load ServiceCall notifications.'
        };
    }

})(
    request,
    response
);
