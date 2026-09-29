(function process(request, response) {

    try {

        /* =====================================================
           TABLES
        ===================================================== */

        var CONVERSATION_TABLE =
            'x_1806573_servic_0_servicecall_conversation';

        var MEMBER_TABLE =
            'x_1806573_servic_0_servicecall_conversation_member';

        var MESSAGE_TABLE =
            'x_1806573_servic_0_servicecall_message';

        var SERVICECALL_USER_TABLE =
            'x_1806573_servic_0_servicecall_user';


        /* =====================================================
           AUTHENTICATED USER
        ===================================================== */

        var currentUserSysId =
            String(
                gs.getUserID() || ''
            );


        if (!currentUserSysId) {

            response.setStatus(401);

            return {
                success: false,
                code: 'NOT_AUTHENTICATED',
                message: 'Authentication required.'
            };
        }


        /* =====================================================
           REQUEST BODY
        ===================================================== */

        var body =
            request.body &&
            request.body.data
                ? request.body.data
                : {};


        var recipientSysId =
            String(
                body.recipient_sys_id || ''
            ).trim();


        var requestedConversationSysId =
            String(
                body.conversation_id || ''
            ).trim();


        var messageText =
            String(
                body.message || ''
            ).trim();


        /*
         * Optional.
         *
         * Empty = normal message.
         * Populated = reply to an existing message.
         */
        var replyToMessageSysId =
            String(
                body.reply_to_message_sys_id || ''
            ).trim();


        /* =====================================================
           MESSAGE VALIDATION
        ===================================================== */

        if (!messageText) {

            response.setStatus(400);

            return {
                success: false,
                code: 'MESSAGE_REQUIRED',
                message: 'Message cannot be empty.'
            };
        }


        if (
            messageText.length >
            10000
        ) {

            response.setStatus(400);

            return {
                success: false,
                code: 'MESSAGE_TOO_LONG',
                message:
                    'Message cannot exceed 10,000 characters.'
            };
        }


        /* =====================================================
           REPLY SYS_ID VALIDATION

           Conversation validation happens later,
           after the target conversation has been
           authoritatively resolved.
        ===================================================== */

        if (
            replyToMessageSysId &&
            !/^[0-9a-f]{32}$/i.test(
                replyToMessageSysId
            )
        ) {

            response.setStatus(400);

            return {
                success: false,
                code: 'INVALID_REPLY_MESSAGE',
                message:
                    'Invalid reply message.'
            };
        }


        /* =====================================================
           VERIFY CURRENT USER
        ===================================================== */

        var currentUserGR =
            new GlideRecord(
                'sys_user'
            );


        if (
            !currentUserGR.get(
                currentUserSysId
            )
        ) {

            response.setStatus(404);

            return {
                success: false,
                code: 'CURRENT_USER_NOT_FOUND',
                message:
                    'Authenticated user was not found.'
            };
        }


        /* =====================================================
           VERIFY CURRENT USER IS ACTIVE SERVICECALL USER
        ===================================================== */

        var currentServiceCallUserGR =
            new GlideRecord(
                SERVICECALL_USER_TABLE
            );


        currentServiceCallUserGR.addQuery(
            'u_user',
            currentUserSysId
        );


        currentServiceCallUserGR.addQuery(
            'u_active',
            true
        );


        currentServiceCallUserGR.setLimit(1);

        currentServiceCallUserGR.query();


        if (
            !currentServiceCallUserGR.next()
        ) {

            response.setStatus(403);

            return {
                success: false,
                code: 'CURRENT_USER_NOT_SERVICECALL_USER',
                message:
                    'You are not an active ServiceCall user.'
            };
        }


        var now =
            new GlideDateTime();


        var conversationGR =
            new GlideRecord(
                CONVERSATION_TABLE
            );


        var conversationSysId =
            '';


        var conversationCreated =
            false;


        var currentMemberSysId =
            '';


        var conversationType =
            '';


        var recipientGR =
            null;


        /* =====================================================
           MODE 1:
           EXISTING CONVERSATION

           Currently used for GROUP messaging.
        ===================================================== */

        if (requestedConversationSysId) {

            if (
                !/^[0-9a-f]{32}$/i.test(
                    requestedConversationSysId
                )
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code: 'INVALID_CONVERSATION',
                    message:
                        'Invalid conversation.'
                };
            }


            if (
                !conversationGR.get(
                    requestedConversationSysId
                )
            ) {

                response.setStatus(404);

                return {
                    success: false,
                    code: 'CONVERSATION_NOT_FOUND',
                    message:
                        'Conversation was not found.'
                };
            }


            if (
                conversationGR.getValue(
                    'u_active'
                ) != '1'
            ) {

                response.setStatus(403);

                return {
                    success: false,
                    code: 'CONVERSATION_INACTIVE',
                    message:
                        'This conversation is no longer active.'
                };
            }


            conversationType =
                String(
                    conversationGR.getValue(
                        'u_type'
                    ) || ''
                );


            /*
             * conversation_id sending remains
             * the existing group-chat path.
             */
            if (
                conversationType !==
                'group'
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code: 'INVALID_CONVERSATION_TYPE',
                    message:
                        'Conversation-based sending is currently supported for group conversations.'
                };
            }


            conversationSysId =
                String(
                    conversationGR.getUniqueValue()
                );


            /* =================================================
               VERIFY CURRENT USER IS ACTIVE GROUP MEMBER
            ================================================= */

            var groupMemberGR =
                new GlideRecord(
                    MEMBER_TABLE
                );


            groupMemberGR.addQuery(
                'u_conversation',
                conversationSysId
            );


            groupMemberGR.addQuery(
                'u_user',
                currentUserSysId
            );


            groupMemberGR.addQuery(
                'u_active',
                true
            );


            groupMemberGR.setLimit(1);

            groupMemberGR.query();


            if (
                !groupMemberGR.next()
            ) {

                response.setStatus(403);

                return {
                    success: false,
                    code: 'NOT_CONVERSATION_MEMBER',
                    message:
                        'You are not an active member of this group.'
                };
            }


            currentMemberSysId =
                String(
                    groupMemberGR.getUniqueValue()
                );
        }


        /* =====================================================
           MODE 2:
           DIRECT MESSAGE

           Preserve existing proven behavior.
        ===================================================== */

        else {

            if (!recipientSysId) {

                response.setStatus(400);

                return {
                    success: false,
                    code: 'RECIPIENT_REQUIRED',
                    message:
                        'Recipient is required.'
                };
            }


            if (
                !/^[0-9a-f]{32}$/i.test(
                    recipientSysId
                )
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code: 'INVALID_RECIPIENT',
                    message:
                        'Invalid recipient.'
                };
            }


            if (
                recipientSysId ===
                currentUserSysId
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code:
                        'SELF_MESSAGE_NOT_ALLOWED',
                    message:
                        'You cannot start a direct conversation with yourself.'
                };
            }


            /* =================================================
               VERIFY RECIPIENT
            ================================================= */

            recipientGR =
                new GlideRecord(
                    'sys_user'
                );


            if (
                !recipientGR.get(
                    recipientSysId
                )
            ) {

                response.setStatus(404);

                return {
                    success: false,
                    code:
                        'RECIPIENT_NOT_FOUND',
                    message:
                        'Recipient was not found.'
                };
            }


            /* =================================================
               VERIFY RECIPIENT SERVICECALL ACCESS
            ================================================= */

            var serviceCallUserGR =
                new GlideRecord(
                    SERVICECALL_USER_TABLE
                );


            serviceCallUserGR.addQuery(
                'u_user',
                recipientSysId
            );


            serviceCallUserGR.addQuery(
                'u_active',
                true
            );


            serviceCallUserGR.setLimit(1);

            serviceCallUserGR.query();


            if (
                !serviceCallUserGR.next()
            ) {

                response.setStatus(403);

                return {
                    success: false,
                    code:
                        'RECIPIENT_NOT_SERVICECALL_USER',
                    message:
                        'Recipient is not an active ServiceCall user.'
                };
            }


            /* =================================================
               DIRECT CONVERSATION KEY
            ================================================= */

            var pair =
                [
                    currentUserSysId,
                    recipientSysId
                ];


            pair.sort();


            var conversationKey =
                pair[0] +
                ':' +
                pair[1];


            /* =================================================
               FIND DIRECT CONVERSATION
            ================================================= */

            conversationGR.addQuery(
                'u_conversation_key',
                conversationKey
            );


            conversationGR.addQuery(
                'u_type',
                'direct'
            );


            conversationGR.addQuery(
                'u_active',
                true
            );


            conversationGR.setLimit(1);

            conversationGR.query();


            if (
                !conversationGR.next()
            ) {

                conversationGR.initialize();


                conversationGR.setValue(
                    'u_user_a',
                    pair[0]
                );


                conversationGR.setValue(
                    'u_user_b',
                    pair[1]
                );


                conversationGR.setValue(
                    'u_conversation_key',
                    conversationKey
                );


                conversationGR.setValue(
                    'u_type',
                    'direct'
                );


                conversationGR.setValue(
                    'u_active',
                    true
                );


                var newConversationSysId =
                    conversationGR.insert();


                if (
                    !newConversationSysId
                ) {

                    throw new Error(
                        'Unable to create conversation.'
                    );
                }


                conversationCreated =
                    true;


                if (
                    !conversationGR.get(
                        newConversationSysId
                    )
                ) {

                    throw new Error(
                        'Unable to reload conversation.'
                    );
                }
            }


            conversationSysId =
                String(
                    conversationGR.getUniqueValue()
                );


            conversationType =
                'direct';


            /* =================================================
               ENSURE DIRECT MEMBER
            ================================================= */

            function ensureMember(
                userSysId
            ) {

                var memberGR =
                    new GlideRecord(
                        MEMBER_TABLE
                    );


                memberGR.addQuery(
                    'u_conversation',
                    conversationSysId
                );


                memberGR.addQuery(
                    'u_user',
                    userSysId
                );


                memberGR.setLimit(1);

                memberGR.query();


                if (
                    memberGR.next()
                ) {

                    if (
                        memberGR.getValue(
                            'u_active'
                        ) != '1'
                    ) {

                        memberGR.setValue(
                            'u_active',
                            true
                        );


                        memberGR.update();
                    }


                    return String(
                        memberGR.getUniqueValue()
                    );
                }


                memberGR.initialize();


                memberGR.setValue(
                    'u_conversation',
                    conversationSysId
                );


                memberGR.setValue(
                    'u_user',
                    userSysId
                );


                memberGR.setValue(
                    'u_role',
                    'member'
                );


                memberGR.setValue(
                    'u_joined_at',
                    now
                );


                memberGR.setValue(
                    'u_active',
                    true
                );


                var memberSysId =
                    memberGR.insert();


                if (
                    !memberSysId
                ) {

                    throw new Error(
                        'Unable to create conversation member.'
                    );
                }


                return String(
                    memberSysId
                );
            }


            currentMemberSysId =
                ensureMember(
                    currentUserSysId
                );


            ensureMember(
                recipientSysId
            );
        }


        /* =====================================================
           VALIDATE REPLY TARGET

           IMPORTANT:
           We validate this only AFTER the actual
           conversation has been resolved.

           This prevents a client from replying to
           a message from another conversation.
        ===================================================== */

        var replyToGR =
            null;


        var replyData =
            null;


        if (replyToMessageSysId) {

            replyToGR =
                new GlideRecord(
                    MESSAGE_TABLE
                );


            if (
                !replyToGR.get(
                    replyToMessageSysId
                )
            ) {

                response.setStatus(404);

                return {
                    success: false,
                    code:
                        'REPLY_MESSAGE_NOT_FOUND',
                    message:
                        'The message you are replying to was not found.'
                };
            }


            var replyConversationSysId =
                String(
                    replyToGR.getValue(
                        'u_conversation'
                    ) || ''
                );


            if (
                replyConversationSysId !==
                conversationSysId
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code:
                        'REPLY_MESSAGE_WRONG_CONVERSATION',
                    message:
                        'You can only reply to a message in the current conversation.'
                };
            }


            var replyDeletedValue =
                String(
                    replyToGR.getValue(
                        'u_deleted'
                    ) || ''
                );


            if (
                replyDeletedValue === '1' ||
                replyDeletedValue === 'true'
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code:
                        'REPLY_MESSAGE_DELETED',
                    message:
                        'You cannot reply to a deleted message.'
                };
            }


            var replySenderSysId =
                String(
                    replyToGR.getValue(
                        'u_sender'
                    ) || ''
                );


            var replySenderName =
                'Unknown User';


            if (replySenderSysId) {

                var replySenderGR =
                    new GlideRecord(
                        'sys_user'
                    );


                if (
                    replySenderGR.get(
                        replySenderSysId
                    )
                ) {

                    replySenderName =
                        String(
                            replySenderGR.getDisplayValue() ||
                            replySenderGR.getValue(
                                'name'
                            ) ||
                            'Unknown User'
                        );
                }
            }


            replyData = {

                sys_id:
                    String(
                        replyToGR.getUniqueValue()
                    ),

                sender_sys_id:
                    replySenderSysId,

                sender_name:
                    replySenderName,

                type:
                    String(
                        replyToGR.getValue(
                            'u_type'
                        ) || 'text'
                    ),

                text:
                    String(
                        replyToGR.getValue(
                            'u_message'
                        ) || ''
                    )
            };
        }


        /* =====================================================
           CREATE MESSAGE

           Shared by DIRECT + GROUP.
        ===================================================== */

        var messageGR =
            new GlideRecord(
                MESSAGE_TABLE
            );


        messageGR.initialize();


        messageGR.setValue(
            'u_conversation',
            conversationSysId
        );


        messageGR.setValue(
            'u_sender',
            currentUserSysId
        );


        messageGR.setValue(
            'u_type',
            'text'
        );


        messageGR.setValue(
            'u_message',
            messageText
        );


        /*
         * Only populate u_reply_to for an
         * actual reply.
         */
        if (replyToMessageSysId) {

            messageGR.setValue(
                'u_reply_to',
                replyToMessageSysId
            );
        }


        messageGR.setValue(
            'u_sent_at',
            now
        );


        messageGR.setValue(
            'u_deleted',
            false
        );


        var messageSysId =
            messageGR.insert();


        if (!messageSysId) {

            throw new Error(
                'Unable to create message.'
            );
        }


        /* =====================================================
           UPDATE CONVERSATION PREVIEW
        ===================================================== */

        var preview =
            messageText;


        if (
            preview.length >
            160
        ) {

            preview =
                preview.substring(
                    0,
                    157
                ) +
                '...';
        }


        conversationGR.setValue(
            'u_last_message_at',
            now
        );


        conversationGR.setValue(
            'u_last_message_preview',
            preview
        );


        conversationGR.update();


        /* =====================================================
           MARK SENDER READ
        ===================================================== */

        if (currentMemberSysId) {

            var senderMemberGR =
                new GlideRecord(
                    MEMBER_TABLE
                );


            if (
                senderMemberGR.get(
                    currentMemberSysId
                )
            ) {

                senderMemberGR.setValue(
                    'u_last_read_at',
                    now
                );


                senderMemberGR.update();
            }
        }


        /* =====================================================
           RESPONSE
        ===================================================== */

        response.setStatus(
            conversationCreated
                ? 201
                : 200
        );


        var conversationResponse = {

            sys_id:
                conversationSysId,

            number:
                String(
                    conversationGR.getDisplayValue(
                        'number'
                    ) || ''
                ),

            type:
                conversationType,

            last_message_at:
                now.getValue(),

            last_message_preview:
                preview
        };


        /* =====================================================
           DIRECT RESPONSE DATA
        ===================================================== */

        if (
            conversationType ===
            'direct' &&
            recipientGR
        ) {

            conversationResponse.recipient = {

                sys_id:
                    recipientSysId,

                name:
                    String(
                        recipientGR.getDisplayValue() ||
                        recipientGR.getValue(
                            'name'
                        ) ||
                        ''
                    ),

                user_name:
                    String(
                        recipientGR.getValue(
                            'user_name'
                        ) || ''
                    )
            };
        }


        /* =====================================================
           GROUP RESPONSE DATA
        ===================================================== */

        if (
            conversationType ===
            'group'
        ) {

            conversationResponse.title =
                String(
                    conversationGR.getValue(
                        'u_title'
                    ) || ''
                );
        }


        /* =====================================================
           MESSAGE RESPONSE
        ===================================================== */

        var messageResponse = {

            sys_id:
                String(
                    messageSysId
                ),

            conversation_sys_id:
                conversationSysId,

            sender_sys_id:
                currentUserSysId,

            sender_name:
                String(
                    currentUserGR.getDisplayValue() ||
                    currentUserGR.getValue(
                        'name'
                    ) ||
                    ''
                ),

            type:
                'text',

            text:
                messageText,

            sent_at:
                now.getValue(),

            deleted:
                false,

            /*
             * null for ordinary messages.
             *
             * Object for replies.
             */
            reply_to:
                replyData
        };


        return {

            success: true,

            conversation_created:
                conversationCreated,

            conversation:
                conversationResponse,

            message:
                messageResponse
        };


    } catch (error) {

        gs.error(
            '[ServiceCall Send Message] ' +
            error
        );


        response.setStatus(500);


        return {
            success: false,
            code: 'SEND_MESSAGE_FAILED',
            message:
                'Unable to send the message.'
        };
    }

})(request, response);
