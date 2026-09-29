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
                code: 'NOT_AUTHENTICATED',
                message: 'No authenticated user.'
            };
        }


        /* -----------------------------------------
           TABLES
        ----------------------------------------- */

        var CONVERSATION_TABLE =
            'x_1806573_servic_0_servicecall_conversation';

        var MEMBER_TABLE =
            'x_1806573_servic_0_servicecall_conversation_member';

        var MESSAGE_TABLE =
            'x_1806573_servic_0_servicecall_message';

        var SERVICECALL_USER_TABLE =
            'x_1806573_servic_0_servicecall_user';


        /* -----------------------------------------
           VERIFY SERVICECALL USER
        ----------------------------------------- */

        var serviceCallUserGR =
            new GlideRecord(
                SERVICECALL_USER_TABLE
            );

        serviceCallUserGR.addQuery(
            'u_user',
            currentUserSysId
        );

        serviceCallUserGR.addQuery(
            'u_active',
            true
        );

        serviceCallUserGR.setLimit(1);
        serviceCallUserGR.query();

        if (!serviceCallUserGR.next()) {

            response.setStatus(403);

            return {
                success: false,
                code: 'NOT_SERVICECALL_USER',
                message: 'Current user is not an active ServiceCall user.'
            };
        }


        /* -----------------------------------------
           HELPER:
           PARSE PERSONALLY HIDDEN MESSAGES

           u_hidden_messages stores a JSON array:

           [
               "message_sys_id_1",
               "message_sys_id_2"
           ]

           Fail closed if malformed.
        ----------------------------------------- */

        function parseHiddenMessages(rawValue) {

            var result = {
                valid: true,
                ids: [],
                lookup: {}
            };

            var raw =
                String(rawValue || '').trim();

            if (!raw) {
                return result;
            }

            try {

                var parsed =
                    JSON.parse(raw);

                if (
                    Object.prototype.toString.call(
                        parsed
                    ) !== '[object Array]'
                ) {

                    result.valid = false;

                    return result;
                }

                for (
                    var i = 0;
                    i < parsed.length;
                    i++
                ) {

                    var messageSysId =
                        String(
                            parsed[i] || ''
                        ).trim();

                    if (!messageSysId) {
                        continue;
                    }

                    if (
                        !result.lookup[
                            messageSysId
                        ]
                    ) {

                        result.lookup[
                            messageSysId
                        ] = true;

                        result.ids.push(
                            messageSysId
                        );
                    }
                }

                return result;

            } catch (parseError) {

                result.valid = false;

                return result;
            }
        }


        /* -----------------------------------------
           FIND MY CONVERSATION MEMBERSHIPS

           Load active + inactive memberships.

           Inactive memberships are exposed only
           for historical GROUP conversations.
        ----------------------------------------- */

        var conversationIds = [];

        var membershipByConversation = {};

        var memberGR =
            new GlideRecord(
                MEMBER_TABLE
            );

        memberGR.addQuery(
            'u_user',
            currentUserSysId
        );

        memberGR.query();


        while (memberGR.next()) {

            var conversationSysId =
                String(
                    memberGR.getValue(
                        'u_conversation'
                    ) || ''
                );


            if (!conversationSysId) {
                continue;
            }


            if (
                conversationIds.indexOf(
                    conversationSysId
                ) === -1
            ) {

                conversationIds.push(
                    conversationSysId
                );
            }


            var hiddenMessagesResult =
                parseHiddenMessages(
                    memberGR.getValue(
                        'u_hidden_messages'
                    )
                );


            /*
             * Do not silently ignore malformed
             * visibility data.
             */
            if (
                hiddenMessagesResult.valid !== true
            ) {

                gs.error(
                    '[ServiceCall Conversations] ' +
                    'Invalid u_hidden_messages JSON ' +
                    'for membership ' +
                    String(
                        memberGR.getUniqueValue()
                    )
                );

                response.setStatus(500);

                return {
                    success: false,
                    code: 'HIDDEN_MESSAGES_DATA_INVALID',
                    message: 'Unable to safely load conversation visibility.'
                };
            }


            membershipByConversation[
                conversationSysId
            ] = {

                active:
                    String(
                        memberGR.getValue(
                            'u_active'
                        )
                    ) === '1',

                role:
                    String(
                        memberGR.getValue(
                            'u_role'
                        ) || 'member'
                    ),

                last_read_at:
                    String(
                        memberGR.getValue(
                            'u_last_read_at'
                        ) || ''
                    ),

                joined_at:
                    String(
                        memberGR.getValue(
                            'u_joined_at'
                        ) || ''
                    ),

                left_at:
                    String(
                        memberGR.getValue(
                            'u_left_at'
                        ) || ''
                    ),

                hidden_message_ids:
                    hiddenMessagesResult.ids,

                hidden_message_lookup:
                    hiddenMessagesResult.lookup
            };
        }


        /* -----------------------------------------
           NO CONVERSATIONS
        ----------------------------------------- */

        if (
            conversationIds.length === 0
        ) {

            return {
                success: true,
                conversations: []
            };
        }


        /* -----------------------------------------
           LOAD CONVERSATIONS

           IMPORTANT:

           We will perform the FINAL sort using
           each user's safe last_message_at.

           Therefore database ordering here is
           not treated as authoritative.
        ----------------------------------------- */

        var conversations = [];

        var conversationGR =
            new GlideRecord(
                CONVERSATION_TABLE
            );

        conversationGR.addQuery(
            'sys_id',
            'IN',
            conversationIds.join(',')
        );

        conversationGR.addQuery(
            'u_active',
            true
        );

        conversationGR.query();


        while (conversationGR.next()) {

            var currentConversationSysId =
                String(
                    conversationGR.getUniqueValue()
                );


            var conversationType =
                String(
                    conversationGR.getValue(
                        'u_type'
                    ) || 'direct'
                );


            /* -------------------------------------
               MY MEMBERSHIP
            ------------------------------------- */

            var myMembership =
                membershipByConversation[
                    currentConversationSysId
                ] || null;


            if (!myMembership) {
                continue;
            }


            var membershipActive =
                myMembership.active === true;


            /*
             * DIRECT:
             * inactive membership stays hidden.
             *
             * GROUP:
             * inactive membership may retain
             * historical access.
             */
            if (
                conversationType === 'direct' &&
                !membershipActive
            ) {

                continue;
            }


            var displayName = '';

            var otherUserSysId = '';

            var otherUserName = '';

            var otherUserUserName = '';


            /* -------------------------------------
               DIRECT CONVERSATION
            ------------------------------------- */

            if (
                conversationType === 'direct'
            ) {

                var otherMemberGR =
                    new GlideRecord(
                        MEMBER_TABLE
                    );

                otherMemberGR.addQuery(
                    'u_conversation',
                    currentConversationSysId
                );

                otherMemberGR.addQuery(
                    'u_user',
                    '!=',
                    currentUserSysId
                );

                otherMemberGR.addQuery(
                    'u_active',
                    true
                );

                otherMemberGR.setLimit(1);

                otherMemberGR.query();


                if (otherMemberGR.next()) {

                    otherUserSysId =
                        String(
                            otherMemberGR.getValue(
                                'u_user'
                            ) || ''
                        );


                    if (otherUserSysId) {

                        var otherUserGR =
                            new GlideRecord(
                                'sys_user'
                            );

                        if (
                            otherUserGR.get(
                                otherUserSysId
                            )
                        ) {

                            otherUserName =
                                String(
                                    otherUserGR
                                    .getDisplayValue() ||
                                    otherUserGR
                                    .getValue(
                                        'name'
                                    ) ||
                                    ''
                                );


                            otherUserUserName =
                                String(
                                    otherUserGR
                                    .getValue(
                                        'user_name'
                                    ) || ''
                                );
                        }
                    }
                }


                displayName =
                    otherUserName ||
                    otherUserUserName ||
                    'Unknown User';
            }


            /* -------------------------------------
               GROUP CONVERSATION
            ------------------------------------- */

            else if (
                conversationType === 'group'
            ) {

                displayName =
                    String(
                        conversationGR.getValue(
                            'u_title'
                        ) || ''
                    );


                if (!displayName) {

                    displayName =
                        'Group Conversation';
                }
            }


            /* -------------------------------------
               MY LAST READ TIME
            ------------------------------------- */

            var myLastReadAt =
                String(
                    myMembership.last_read_at ||
                    ''
                );


            var hiddenMessageLookup =
                myMembership
                    .hidden_message_lookup ||
                {};


            /* -------------------------------------
               SAFE LAST MESSAGE FOR THIS USER

               Rules:

               1. Personally hidden messages:
                  skip completely.

               2. Globally deleted messages:
                  remain visible as
                  "Message deleted".

               3. Historical group:
                  never inspect beyond left_at.

               4. Active conversation:
                  newest visible message wins.
            ------------------------------------- */

            var safeLastMessageAt = '';

            var safeLastMessagePreview = '';


            /*
             * Historical groups require a valid
             * leave boundary.

             * Fail closed if it is missing.
             */
            var historicalBoundary = '';


            if (
                conversationType === 'group' &&
                !membershipActive
            ) {

                historicalBoundary =
                    String(
                        myMembership.left_at ||
                        ''
                    );


                if (!historicalBoundary) {

                    /*
                     * Do not inspect any messages.
                     */
                    safeLastMessageAt = '';
                    safeLastMessagePreview = '';
                }
            }


            /*
             * Active memberships may query normally.

             * Inactive group memberships may query
             * only when left_at exists.
             */
            var mayInspectMessages =
                membershipActive ||
                (
                    conversationType === 'group' &&
                    !membershipActive &&
                    !!historicalBoundary
                );


            if (mayInspectMessages) {

                var latestMessageGR =
                    new GlideRecord(
                        MESSAGE_TABLE
                    );


                latestMessageGR.addQuery(
                    'u_conversation',
                    currentConversationSysId
                );


                if (
                    conversationType === 'group' &&
                    !membershipActive
                ) {

                    latestMessageGR.addQuery(
                        'u_sent_at',
                        '<=',
                        historicalBoundary
                    );
                }


                latestMessageGR.orderByDesc(
                    'u_sent_at'
                );


                /*
                 * Deterministic tie-breakers.
                 */
                latestMessageGR.orderByDesc(
                    'sys_created_on'
                );

                latestMessageGR.orderByDesc(
                    'sys_id'
                );


                latestMessageGR.query();


                /*
                 * We deliberately scan until the
                 * first message visible to THIS
                 * membership is found.

                 * We do not use setLimit(1),
                 * because the newest database row
                 * may be personally hidden.
                 */
                while (
                    latestMessageGR.next()
                ) {

                    var latestMessageSysId =
                        String(
                            latestMessageGR
                            .getUniqueValue()
                        );


                    /*
                     * DELETE FOR ME:
                     *
                     * Invisible only to this
                     * membership.
                     */
                    if (
                        hiddenMessageLookup[
                            latestMessageSysId
                        ]
                    ) {

                        continue;
                    }


                    safeLastMessageAt =
                        String(
                            latestMessageGR.getValue(
                                'u_sent_at'
                            ) || ''
                        );


                    var latestMessageDeleted =
                        String(
                            latestMessageGR.getValue(
                                'u_deleted'
                            )
                        ) === '1';


                    if (latestMessageDeleted) {

                        safeLastMessagePreview =
                            'Message deleted';

                    } else {

                        safeLastMessagePreview =
                            String(
                                latestMessageGR
                                .getValue(
                                    'u_message'
                                ) || ''
                            );
                    }


                    /*
                     * First visible message is
                     * authoritative for this user.
                     */
                    break;
                }
            }


            /* -------------------------------------
               CALCULATE UNREAD COUNT

               Active membership only.

               IMPORTANT:
               Personally hidden messages must not
               contribute to unread count.
            ------------------------------------- */

            var unreadCount = 0;


            if (membershipActive) {

                var unreadMessageGR =
                    new GlideRecord(
                        MESSAGE_TABLE
                    );


                unreadMessageGR.addQuery(
                    'u_conversation',
                    currentConversationSysId
                );


                unreadMessageGR.addQuery(
                    'u_sender',
                    '!=',
                    currentUserSysId
                );


                /*
                 * Preserve the existing behavior:
                 * globally deleted messages are
                 * not unread messages.
                 */
                unreadMessageGR.addQuery(
                    'u_deleted',
                    false
                );


                if (myLastReadAt) {

                    unreadMessageGR.addQuery(
                        'u_sent_at',
                        '>',
                        myLastReadAt
                    );
                }


                unreadMessageGR.query();


                while (
                    unreadMessageGR.next()
                ) {

                    var unreadMessageSysId =
                        String(
                            unreadMessageGR
                            .getUniqueValue()
                        );


                    /*
                     * DELETE FOR ME:
                     * do not count hidden messages.
                     */
                    if (
                        hiddenMessageLookup[
                            unreadMessageSysId
                        ]
                    ) {

                        continue;
                    }


                    unreadCount++;
                }
            }


            /* -------------------------------------
               RESPONSE OBJECT
            ------------------------------------- */

            conversations.push({

                sys_id:
                    currentConversationSysId,

                number:
                    String(
                        conversationGR
                        .getDisplayValue(
                            'number'
                        ) || ''
                    ),

                type:
                    conversationType,

                title:
                    String(
                        conversationGR.getValue(
                            'u_title'
                        ) || ''
                    ),

                display_name:
                    displayName,

                other_user_sys_id:
                    otherUserSysId,

                other_user_name:
                    otherUserName,

                other_user_user_name:
                    otherUserUserName,

                /*
                 * These are now calculated from
                 * the newest message visible to
                 * THIS membership.
                 */
                last_message_at:
                    safeLastMessageAt,

                last_message_preview:
                    safeLastMessagePreview,


                /* ---------------------------------
                   MEMBERSHIP STATE
                --------------------------------- */

                membership_active:
                    membershipActive,

                membership_role:
                    String(
                        myMembership.role ||
                        'member'
                    ),

                joined_at:
                    String(
                        myMembership.joined_at ||
                        ''
                    ),


                /* ---------------------------------
                   READ / UNREAD
                --------------------------------- */

                unread_count:
                    unreadCount,

                last_read_at:
                    myLastReadAt
            });
        }


        /* -----------------------------------------
           FINAL USER-SPECIFIC SORT

           We cannot safely use the conversation's
           global u_last_message_at because:

           - Delete for me is membership-specific.
           - Historical group members have a
             personal leave boundary.

           Therefore sort using the safe timestamp
           returned for this user.
        ----------------------------------------- */

        conversations.sort(
            function(a, b) {

                var aTime =
                    String(
                        a.last_message_at || ''
                    );

                var bTime =
                    String(
                        b.last_message_at || ''
                    );


                if (
                    aTime === bTime
                ) {

                    /*
                     * Stable deterministic fallback.
                     */
                    var aNumber =
                        String(
                            a.number || ''
                        );

                    var bNumber =
                        String(
                            b.number || ''
                        );


                    if (
                        aNumber < bNumber
                    ) {
                        return -1;
                    }


                    if (
                        aNumber > bNumber
                    ) {
                        return 1;
                    }


                    return 0;
                }


                /*
                 * ServiceNow internal date/time
                 * values are lexically sortable:
                 *
                 * yyyy-MM-dd HH:mm:ss
                 */
                return (
                    aTime > bTime
                        ? -1
                        : 1
                );
            }
        );


        /* -----------------------------------------
           SUCCESS
        ----------------------------------------- */

        return {
            success: true,
            conversations: conversations
        };


    } catch (ex) {

        gs.error(
            '[ServiceCall Conversations] ' +
            ex.message
        );


        response.setStatus(500);


        return {
            success: false,
            code: 'SERVER_ERROR',
            message: 'Unable to load conversations.'
        };
    }

})(request, response);
