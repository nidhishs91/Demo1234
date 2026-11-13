Invite Participant - 
(function process(
    request,
    response
) {

    var result = {
        success: false
    };


    try {

        /* -----------------------------------------------
           AUTHENTICATED USER
        ----------------------------------------------- */

        var currentUserId =
            gs.getUserID();


        if (!currentUserId) {

            response.setStatus(401);

            result.code =
                'UNAUTHORIZED';

            result.message =
                'Authenticated ServiceCall user was not found.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           REQUEST BODY
        ----------------------------------------------- */

        var body =
            request.body.data || {};


        var callSysId =
            String(
                body.call_sys_id || ''
            ).trim();


        var targetUserId =
            String(
                body.user_sys_id || ''
            ).trim();


        if (!callSysId) {

            response.setStatus(400);

            result.code =
                'CALL_ID_REQUIRED';

            result.message =
                'Call ID was not provided.';

            response.setBody(result);
            return;
        }


        if (!targetUserId) {

            response.setStatus(400);

            result.code =
                'USER_ID_REQUIRED';

            result.message =
                'User to invite was not provided.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           GET EXISTING CALL
        ----------------------------------------------- */

        var callGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_call'
            );


        if (!callGR.get(callSysId)) {

            response.setStatus(404);

            result.code =
                'CALL_NOT_FOUND';

            result.message =
                'ServiceCall call was not found.';

            response.setBody(result);
            return;
        }


        var callState =
            callGR.getValue(
                'u_state'
            ) || '';


        if (
            callState !== 'connected' &&
            callState !== 'ringing'
        ) {

            response.setStatus(409);

            result.code =
                'CALL_NOT_ACTIVE';

            result.message =
                'Participants can only be added to an active call.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           REQUESTER MUST BELONG TO THIS CALL
        ----------------------------------------------- */

        var requesterParticipant =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_participant'
            );


        requesterParticipant.addQuery(
            'u_call',
            callSysId
        );

        requesterParticipant.addQuery(
            'u_user',
            currentUserId
        );

        requesterParticipant.addQuery(
            'u_status',
            'IN',
            'invited,ringing,connected'
        );

        requesterParticipant.setLimit(1);
        requesterParticipant.query();


        if (
            !requesterParticipant.next()
        ) {

            response.setStatus(403);

            result.code =
                'NOT_CALL_PARTICIPANT';

            result.message =
                'You are not an active participant in this call.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           TARGET USER
        ----------------------------------------------- */

        if (
            targetUserId ===
            currentUserId
        ) {

            response.setStatus(409);

            result.code =
                'CANNOT_INVITE_SELF';

            result.message =
                'You are already participating in this call.';

            response.setBody(result);
            return;
        }


        var userGR =
            new GlideRecord(
                'sys_user'
            );


        if (
            !userGR.get(
                targetUserId
            ) ||
            !userGR.getValue(
                'active'
            )
        ) {

            response.setStatus(404);

            result.code =
                'USER_NOT_AVAILABLE';

            result.message =
                'The selected user is not available.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           PREVENT DUPLICATE PARTICIPANT
        ----------------------------------------------- */

        var existingParticipant =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_participant'
            );


        existingParticipant.addQuery(
            'u_call',
            callSysId
        );

        existingParticipant.addQuery(
            'u_user',
            targetUserId
        );

        existingParticipant.addQuery(
            'u_status',
            'IN',
            'invited,ringing,connected'
        );

        existingParticipant.setLimit(1);
        existingParticipant.query();


        if (
            existingParticipant.next()
        ) {

            response.setStatus(409);

            result.code =
                'USER_ALREADY_IN_CALL';

            result.message =
                'The selected user is already part of this call.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           DESKTOP READINESS

           Same readiness rules already used by ServiceCall:
           connected + active + notifications enabled +
           recent heartbeat.
        ----------------------------------------------- */

        var registrationGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_desktop_registration'
            );


        registrationGR.addQuery(
            'u_user',
            targetUserId
        );

        registrationGR.addQuery(
            'u_status',
            'connected'
        );

        registrationGR.addQuery(
            'is_this_active',
            true
        );

        registrationGR.addQuery(
            'u_notifications',
            true
        );


        var heartbeatThreshold =
            new GlideDateTime();

        heartbeatThreshold.addSeconds(
            -60
        );


        registrationGR.addQuery(
            'u_last_seen',
            '>=',
            heartbeatThreshold
        );


        /*
         * Prevent a future timestamp from
         * accidentally passing readiness.
         */
        registrationGR.addQuery(
            'u_last_seen',
            '<=',
            new GlideDateTime()
        );


        registrationGR.setLimit(1);
        registrationGR.query();


        if (
            !registrationGR.next()
        ) {

            response.setStatus(409);

            result.code =
                'TARGET_DESKTOP_NOT_READY';

            result.message =
                'The selected user\'s ServiceCall Desktop is not connected.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           CREATE PARTICIPANT

           IMPORTANT:
           We DO NOT create another Call record.

           This participant belongs to the existing
           call and therefore uses the existing
           u_conference_id.
        ----------------------------------------------- */

        var participantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_participant'
            );


        participantGR.initialize();


        participantGR.u_call =
            callSysId;

        participantGR.u_user =
            targetUserId;

        participantGR.u_role =
            'participant';

        participantGR.u_status =
            'ringing';

        participantGR.u_is_owner =
            false;


        var participantSysId =
            participantGR.insert();


        if (!participantSysId) {

            response.setStatus(500);

            result.code =
                'PARTICIPANT_CREATE_FAILED';

            result.message =
                'Unable to add the participant to the call.';

            response.setBody(result);
            return;
        }


        /* -----------------------------------------------
           SUCCESS
        ----------------------------------------------- */

        result.success =
            true;

        result.code =
            'PARTICIPANT_INVITED';

        result.message =
            'Participant invited successfully.';

        result.call_sys_id =
            callSysId;

        result.participant_sys_id =
            String(
                participantSysId
            );

        result.user_sys_id =
            targetUserId;

        result.user_name =
            userGR.getDisplayValue();

        result.conference_id =
            callGR.getValue(
                'u_conference_id'
            ) || '';


        response.setStatus(200);
        response.setBody(result);


    } catch (error) {

        gs.error(
            'ServiceCall Invite Participant error: ' +
            error.message
        );


        response.setStatus(500);


        response.setBody({

            success: false,

            code:
                'INVITE_PARTICIPANT_FAILED',

            message:
                'Unable to invite the participant.'

        });
    }

})(

Start Meeting - 
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
           REQUEST BODY
        ----------------------------------------- */

        var body =
            request.body.data || {};

        var meetingSysId =
            String(
                body.meeting_sys_id || ''
            ).trim();


        if (!meetingSysId) {

            response.setStatus(400);

            return {
                success: false,
                code: 'MEETING_REQUIRED',
                message: 'Meeting sys_id is required.'
            };
        }


        /* -----------------------------------------
           GET MEETING
        ----------------------------------------- */

        var meetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );

        if (!meetingGR.get(meetingSysId)) {

            response.setStatus(404);

            return {
                success: false,
                code: 'MEETING_NOT_FOUND',
                message: 'Meeting was not found.'
            };
        }


        /* -----------------------------------------
           CHECK MEETING STATE
        ----------------------------------------- */

        var meetingState =
            String(
                meetingGR.getValue('u_state') || ''
            );


        if (meetingState === 'cancelled') {

            response.setStatus(409);

            return {
                success: false,
                code: 'MEETING_CANCELLED',
                message: 'This meeting has been cancelled.'
            };
        }


        if (meetingState === 'ended') {

            response.setStatus(409);

            return {
                success: false,
                code: 'MEETING_ENDED',
                message: 'This meeting has already ended.'
            };
        }


        /*
         * Prevent creating another live call
         * if Start is pressed more than once.
         */
        if (meetingState === 'in progress') {

            response.setStatus(200);

            return {
                success: true,
                code: 'MEETING_ALREADY_STARTED',
                message: 'This meeting is already in progress.',

                meeting_sys_id: meetingGR.getUniqueValue(),

                meeting_number: meetingGR.getDisplayValue(
                    'number'
                ),

                title: meetingGR.getValue(
                    'u_title'
                ),

                started_by: meetingGR.getValue(
                    'u_started_by'
                ),

                started_by_name: meetingGR.getDisplayValue(
                    'u_started_by'
                ),

                started_at: meetingGR.getValue(
                    'u_started_at'
                ),

                call_sys_id: meetingGR.getValue(
                    'u_created_call'
                ),

                conference_id: meetingGR.getValue(
                    'u_conference_id'
                )
            };
        }


        /* -----------------------------------------
           VERIFY USER BELONGS TO MEETING
        ----------------------------------------- */

        var meetingParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting_participant'
            );

        meetingParticipantGR.addQuery(
            'u_meeting',
            meetingSysId
        );

        meetingParticipantGR.addQuery(
            'u_user',
            currentUserSysId
        );

        meetingParticipantGR.setLimit(1);

        meetingParticipantGR.query();


        if (!meetingParticipantGR.next()) {

            response.setStatus(403);

            return {
                success: false,
                code: 'NOT_MEETING_PARTICIPANT',
                message: 'You are not a participant of this meeting.'
            };
        }


        /* -----------------------------------------
           GET CONFERENCE ID
        ----------------------------------------- */

        var conferenceId =
            String(
                meetingGR.getValue(
                    'u_conference_id'
                ) || ''
            );


        /*
         * Normally this already exists because
         * /create-meeting created it.
         *
         * Keep this fallback for safety.
         */
        if (!conferenceId) {

            conferenceId =
                'servicecall_meeting_' +
                gs.generateGUID()
                .replace(
                    /-/g,
                    ''
                );

            meetingGR.setValue(
                'u_conference_id',
                conferenceId
            );
        }


        /* -----------------------------------------
           CREATE LIVE SERVICECALL CALL
        ----------------------------------------- */

        var callGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_call'
            );

        callGR.initialize();

        callGR.setValue(
            'u_state',
            'connected'
        );

        callGR.setValue(
            'u_original_caller',
            currentUserSysId
        );

        callGR.setValue(
            'u_started_at',
            new GlideDateTime()
        );

        callGR.setValue(
            'u_conference_id',
            conferenceId
        );


        var callSysId =
            callGR.insert();


        if (!callSysId) {

            throw new Error(
                'Unable to create live call.'
            );
        }


        /* -----------------------------------------
           CREATE LIVE PARTICIPANT FOR STARTER
        ----------------------------------------- */

        var liveParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_participant'
            );

        liveParticipantGR.initialize();

        liveParticipantGR.setValue(
            'u_call',
            callSysId
        );

        liveParticipantGR.setValue(
            'u_user',
            currentUserSysId
        );

        liveParticipantGR.setValue(
            'u_role',
            'caller'
        );

        liveParticipantGR.setValue(
            'u_status',
            'connected'
        );

        liveParticipantGR.setValue(
            'u_is_owner',
            true
        );

        liveParticipantGR.setValue(
            'u_joined_at',
            new GlideDateTime()
        );

        liveParticipantGR.insert();


        /* -----------------------------------------
           UPDATE STARTER'S MEETING PARTICIPANT
        ----------------------------------------- */

        meetingParticipantGR.setValue(
            'u_invitation_status',
            'accepted'
        );

        meetingParticipantGR.setValue(
            'u_join_status',
            'joined'
        );

        meetingParticipantGR.setValue(
            'u_joined_at',
            new GlideDateTime()
        );

        meetingParticipantGR.update();


        /* -----------------------------------------
           UPDATE MEETING
        ----------------------------------------- */

        var startedAt =
            new GlideDateTime();

        meetingGR.setValue(
            'u_state',
            'in progress'
        );

        meetingGR.setValue(
            'u_started_by',
            currentUserSysId
        );

        meetingGR.setValue(
            'u_started_at',
            startedAt
        );

        meetingGR.setValue(
            'u_created_call',
            callSysId
        );

        meetingGR.update();


        /* -----------------------------------------
           CREATE MEETING STARTED NOTIFICATIONS
        ----------------------------------------- */

        var notifyParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting_participant'
            );

        notifyParticipantGR.addQuery(
            'u_meeting',
            meetingSysId
        );


        /*
         * IMPORTANT:
         *
         * Do NOT notify the person who just
         * started the meeting.
         *
         * That user is already entering the
         * live meeting and should not receive
         * their own ringing Meeting Started
         * notification.
         */
        notifyParticipantGR.addQuery(
            'u_user',
            '!=',
            currentUserSysId
        );


        /*
         * Do not notify somebody who
         * explicitly declined the meeting.
         */
        notifyParticipantGR.addQuery(
            'u_invitation_status',
            '!=',
            'declined'
        );

        notifyParticipantGR.query();


        while (
            notifyParticipantGR.next()
        ) {

            var notifyUserSysId =
                String(
                    notifyParticipantGR.getValue(
                        'u_user'
                    ) || ''
                );


            if (!notifyUserSysId) {
                continue;
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
                notifyUserSysId
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
                continue;
            }


            var serviceCallUserSysId =
                String(
                    serviceCallUserGR.getUniqueValue()
                );


            /* -----------------------------------------
               DUPLICATE PROTECTION
            ----------------------------------------- */

            var existingStartedNotificationGR =
                new GlideRecord(
                    'x_1806573_servic_0_servicecall_notification'
                );

            existingStartedNotificationGR.addQuery(
                'u_recipient',
                serviceCallUserSysId
            );

            existingStartedNotificationGR.addQuery(
                'u_meeting',
                meetingSysId
            );

            existingStartedNotificationGR.addQuery(
                'u_type',
                'meeting started'
            );

            existingStartedNotificationGR.setLimit(
                1
            );

            existingStartedNotificationGR.query();


            if (
                existingStartedNotificationGR.next()
            ) {
                continue;
            }


            /* -----------------------------------------
               CREATE NOTIFICATION
            ----------------------------------------- */

            var startedNotificationGR =
                new GlideRecord(
                    'x_1806573_servic_0_servicecall_notification'
                );

            startedNotificationGR.initialize();

            startedNotificationGR.setValue(
                'u_recipient',
                serviceCallUserSysId
            );

            startedNotificationGR.setValue(
                'u_type',
                'meeting started'
            );

            startedNotificationGR.setValue(
                'u_title',
                'Meeting started'
            );

            startedNotificationGR.setValue(
                'u_message',
                '"' +
                String(
                    meetingGR.getValue(
                        'u_title'
                    ) ||
                    'ServiceCall meeting'
                ) +
                '" has started.'
            );

            startedNotificationGR.setValue(
                'u_meeting',
                meetingSysId
            );

            startedNotificationGR.setValue(
                'u_call',
                callSysId
            );

            startedNotificationGR.setValue(
                'u_read',
                false
            );

            startedNotificationGR.setValue(
                'u_action_type',
                'open meeting'
            );

            startedNotificationGR.setValue(
                'u_priority',
                'normal'
            );

            startedNotificationGR.insert();
        }


        /* -----------------------------------------
           RESPONSE
        ----------------------------------------- */

        response.setStatus(200);

        return {

            success: true,

            code: 'MEETING_STARTED',

            meeting_sys_id: meetingGR.getUniqueValue(),

            meeting_number: meetingGR.getDisplayValue(
                'number'
            ),

            title: meetingGR.getValue(
                'u_title'
            ),

            started_by: currentUserSysId,

            started_by_name: gs.getUserDisplayName(),

            started_at: meetingGR.getValue(
                'u_started_at'
            ),

            call_sys_id: String(callSysId),

            call_number: callGR.getDisplayValue(
                'number'
            ),

            conference_id: conferenceId
        };


    } catch (error) {

        gs.error(
            'ServiceCall Start Meeting failed: ' +
            error.message
        );

        response.setStatus(500);

        return {
            success: false,
            code: 'MEETING_START_FAILED',
            message: 'Unable to start the ServiceCall meeting.'
        };
    }

})(
    request,
    response
);

Join Meeting - 
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
           REQUEST
        ----------------------------------------- */
 
        var body =
            request.body.data || {};
 
        var meetingSysId =
            String(
                body.meeting_sys_id || ''
            ).trim();
 
 
        if (!meetingSysId) {
 
            response.setStatus(400);
 
            return {
                success: false,
                code: 'MEETING_REQUIRED',
                message: 'Meeting sys_id is required.'
            };
        }
 
 
        /* -----------------------------------------
           GET MEETING
        ----------------------------------------- */
 
        var meetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );
 
        if (!meetingGR.get(meetingSysId)) {
 
            response.setStatus(404);
 
            return {
                success: false,
                code: 'MEETING_NOT_FOUND',
                message: 'Meeting was not found.'
            };
        }
 
 
        /* -----------------------------------------
           MEETING MUST BE LIVE
        ----------------------------------------- */
 
        var meetingState =
            String(
                meetingGR.getValue(
                    'u_state'
                ) || ''
            );
 
        if (meetingState !== 'in progress') {
 
            response.setStatus(409);
 
            return {
                success: false,
                code: 'MEETING_NOT_IN_PROGRESS',
                message:
                    'This meeting is not currently in progress.'
            };
        }
 
 
        /* -----------------------------------------
           VERIFY MEETING PARTICIPANT
        ----------------------------------------- */
 
        var meetingParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting_participant'
            );
 
        meetingParticipantGR.addQuery(
            'u_meeting',
            meetingSysId
        );
 
        meetingParticipantGR.addQuery(
            'u_user',
            currentUserSysId
        );
 
        meetingParticipantGR.setLimit(1);
 
        meetingParticipantGR.query();
 
 
        if (!meetingParticipantGR.next()) {
 
            response.setStatus(403);
 
            return {
                success: false,
                code: 'NOT_MEETING_PARTICIPANT',
                message:
                    'You are not currently invited to this meeting.'
            };
        }
 
 
        if (
            String(
                meetingParticipantGR.getValue(
                    'u_invitation_status'
                ) || ''
            ) === 'declined'
        ) {
 
            response.setStatus(403);
 
            return {
                success: false,
                code: 'MEETING_INVITATION_DECLINED',
                message:
                    'This meeting invitation was declined.'
            };
        }
 
 
        /* -----------------------------------------
           GET LIVE CALL
        ----------------------------------------- */
 
        var callSysId =
            String(
                meetingGR.getValue(
                    'u_created_call'
                ) || ''
            );
 
        if (!callSysId) {
 
            response.setStatus(409);
 
            return {
                success: false,
                code: 'MEETING_CALL_NOT_FOUND',
                message:
                    'The live meeting call is not available.'
            };
        }
 
 
        var callGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_call'
            );
 
        if (!callGR.get(callSysId)) {
 
            response.setStatus(404);
 
            return {
                success: false,
                code: 'CALL_NOT_FOUND',
                message:
                    'The ServiceCall conference was not found.'
            };
        }
 
 
        if (
            String(
                callGR.getValue(
                    'u_state'
                ) || ''
            ) === 'ended'
        ) {
 
            response.setStatus(409);
 
            return {
                success: false,
                code: 'CALL_ENDED',
                message:
                    'The meeting call has already ended.'
            };
        }
 
 
        /* -----------------------------------------
           CHECK EXISTING LIVE PARTICIPANT
        ----------------------------------------- */
 
        var liveParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_participant'
            );
 
        liveParticipantGR.addQuery(
            'u_call',
            callSysId
        );
 
        liveParticipantGR.addQuery(
            'u_user',
            currentUserSysId
        );
 
        liveParticipantGR.setLimit(1);
 
        liveParticipantGR.query();
 
 
        var joinedAt =
            new GlideDateTime();
 
 
        if (liveParticipantGR.next()) {
 
            /*
             * Handles rejoining the same meeting.
             */
            liveParticipantGR.setValue(
                'u_status',
                'connected'
            );
 
            liveParticipantGR.setValue(
                'u_joined_at',
                joinedAt
            );
 
            liveParticipantGR.setValue(
                'u_left_at',
                ''
            );
 
            liveParticipantGR.update();
 
        } else {
 
            /* -----------------------------------------
               CREATE LIVE PARTICIPANT
            ----------------------------------------- */
 
            liveParticipantGR.initialize();
 
            liveParticipantGR.setValue(
                'u_call',
                callSysId
            );
 
            liveParticipantGR.setValue(
                'u_user',
                currentUserSysId
            );
 
            liveParticipantGR.setValue(
                'u_role',
                'participant'
            );
 
            liveParticipantGR.setValue(
                'u_status',
                'connected'
            );
 
            liveParticipantGR.setValue(
                'u_is_owner',
                false
            );
 
            liveParticipantGR.setValue(
                'u_joined_at',
                joinedAt
            );
 
            liveParticipantGR.insert();
        }
 
 
        /* -----------------------------------------
           UPDATE MEETING PARTICIPANT
        ----------------------------------------- */
 
        meetingParticipantGR.setValue(
            'u_invitation_status',
            'accepted'
        );
 
        meetingParticipantGR.setValue(
            'u_join_status',
            'joined'
        );
 
        meetingParticipantGR.setValue(
            'u_joined_at',
            joinedAt
        );
 
        meetingParticipantGR.setValue(
            'u_left_at',
            ''
        );
 
        meetingParticipantGR.update();
 
 
        /* -----------------------------------------
           RESPONSE
        ----------------------------------------- */
 
        response.setStatus(200);
 
        return {
 
            success: true,
 
            code: 'MEETING_JOINED',
 
            meeting_sys_id:
                meetingGR.getUniqueValue(),
 
            meeting_number:
                meetingGR.getDisplayValue(
                    'number'
                ),
 
            title:
                String(
                    meetingGR.getValue(
                        'u_title'
                    ) || ''
                ),
 
            call_sys_id:
                callSysId,
 
            call_number:
                callGR.getDisplayValue(
                    'number'
                ),
 
            conference_id:
                String(
                    meetingGR.getValue(
                        'u_conference_id'
                    ) || ''
                ),
 
            participant_sys_id:
                liveParticipantGR.getUniqueValue(),
 
            joined_at:
                meetingParticipantGR.getValue(
                    'u_joined_at'
                ),
 
            organizer_name:
                meetingGR.getDisplayValue(
                    'u_organizer'
                ),
 
            started_by_name:
                meetingGR.getDisplayValue(
                    'u_started_by'
                )
        };
 
 
    } catch (error) {
 
        gs.error(
            'ServiceCall Join Meeting failed: ' +
            error.message
        );
 
        response.setStatus(500);
 
        return {
            success: false,
            code: 'MEETING_JOIN_FAILED',
            message:
                'Unable to join the ServiceCall meeting.'
        };
    }
 
})(
    request,
    response
);

Leave Meeting - 
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
           REQUEST
        ----------------------------------------- */
 
        var body =
            request.body.data || {};
 
        var meetingSysId =
            String(
                body.meeting_sys_id || ''
            ).trim();
 
 
        if (!meetingSysId) {
 
            response.setStatus(400);
 
            return {
                success: false,
                code: 'MEETING_REQUIRED',
                message: 'Meeting sys_id is required.'
            };
        }
 
 
        /* -----------------------------------------
           GET MEETING
        ----------------------------------------- */
 
        var meetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );
 
        if (!meetingGR.get(meetingSysId)) {
 
            response.setStatus(404);
 
            return {
                success: false,
                code: 'MEETING_NOT_FOUND',
                message: 'Meeting was not found.'
            };
        }
 
 
        /* -----------------------------------------
           MEETING MUST BE IN PROGRESS
        ----------------------------------------- */
 
        var meetingState =
            String(
                meetingGR.getValue(
                    'u_state'
                ) || ''
            );
 
        if (meetingState !== 'in progress') {
 
            response.setStatus(409);
 
            return {
                success: false,
                code: 'MEETING_NOT_IN_PROGRESS',
                message:
                    'This meeting is not currently in progress.'
            };
        }
 
 
        /* -----------------------------------------
           FIND MEETING PARTICIPANT
        ----------------------------------------- */
 
        var meetingParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting_participant'
            );
 
        meetingParticipantGR.addQuery(
            'u_meeting',
            meetingSysId
        );
 
        meetingParticipantGR.addQuery(
            'u_user',
            currentUserSysId
        );
 
        meetingParticipantGR.setLimit(1);
        meetingParticipantGR.query();
 
 
        if (!meetingParticipantGR.next()) {
 
            response.setStatus(403);
 
            return {
                success: false,
                code: 'NOT_MEETING_PARTICIPANT',
                message:
                    'You are not a participant of this meeting.'
            };
        }
 
 
        /* -----------------------------------------
           USER MUST CURRENTLY BE JOINED
        ----------------------------------------- */
 
        var joinStatus =
            String(
                meetingParticipantGR.getValue(
                    'u_join_status'
                ) || ''
            );
 
        if (joinStatus !== 'joined') {
 
            response.setStatus(409);
 
            return {
                success: false,
                code: 'NOT_CURRENTLY_JOINED',
                message:
                    'You are not currently joined to this meeting.'
            };
        }
 
 
        /* -----------------------------------------
           GET LIVE CALL
        ----------------------------------------- */
 
        var callSysId =
            String(
                meetingGR.getValue(
                    'u_created_call'
                ) || ''
            );
 
        if (!callSysId) {
 
            response.setStatus(409);
 
            return {
                success: false,
                code: 'MEETING_CALL_NOT_FOUND',
                message:
                    'The live meeting call is not available.'
            };
        }
 
 
        var callGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_call'
            );
 
        if (!callGR.get(callSysId)) {
 
            response.setStatus(404);
 
            return {
                success: false,
                code: 'CALL_NOT_FOUND',
                message:
                    'The ServiceCall conference was not found.'
            };
        }
 
 
        /* -----------------------------------------
           FIND LIVE PARTICIPANT
        ----------------------------------------- */
 
        var liveParticipantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_participant'
            );
 
        liveParticipantGR.addQuery(
            'u_call',
            callSysId
        );
 
        liveParticipantGR.addQuery(
            'u_user',
            currentUserSysId
        );
 
        liveParticipantGR.setLimit(1);
        liveParticipantGR.query();
 
 
        if (!liveParticipantGR.next()) {
 
            response.setStatus(404);
 
            return {
                success: false,
                code: 'LIVE_PARTICIPANT_NOT_FOUND',
                message:
                    'Your live meeting participant record was not found.'
            };
        }
 
 
        /* -----------------------------------------
           LEAVE TIME
        ----------------------------------------- */
 
        var leftAt =
            new GlideDateTime();
 
 
        /* -----------------------------------------
           UPDATE LIVE PARTICIPANT
        ----------------------------------------- */
 
        liveParticipantGR.setValue(
            'u_status',
            'left'
        );
 
        liveParticipantGR.setValue(
            'u_left_at',
            leftAt
        );
 
        liveParticipantGR.update();
 
 
        /* -----------------------------------------
           UPDATE MEETING PARTICIPANT
        ----------------------------------------- */
 
        meetingParticipantGR.setValue(
            'u_join_status',
            'left'
        );
 
        meetingParticipantGR.setValue(
            'u_left_at',
            leftAt
        );
 
        meetingParticipantGR.update();
 
 
        /* -----------------------------------------
           DO NOT END MEETING OR CALL
        ----------------------------------------- */
 
        response.setStatus(200);
 
        return {
 
            success: true,
 
            code: 'MEETING_LEFT',
 
            meeting_sys_id:
                meetingGR.getUniqueValue(),
 
            meeting_number:
                meetingGR.getDisplayValue(
                    'number'
                ),
 
            title:
                String(
                    meetingGR.getValue(
                        'u_title'
                    ) || ''
                ),
 
            call_sys_id:
                callSysId,
 
            call_number:
                callGR.getDisplayValue(
                    'number'
                ),
 
            user_sys_id:
                currentUserSysId,
 
            left_at:
                meetingParticipantGR.getValue(
                    'u_left_at'
                ),
 
            meeting_state:
                String(
                    meetingGR.getValue(
                        'u_state'
                    ) || ''
                ),
 
            call_state:
                String(
                    callGR.getValue(
                        'u_state'
                    ) || ''
                )
        };
 
 
    } catch (error) {
 
        gs.error(
            'ServiceCall Leave Meeting failed: ' +
            error.message
        );
 
        response.setStatus(500);
 
        return {
            success: false,
            code: 'MEETING_LEAVE_FAILED',
            message:
                'Unable to leave the ServiceCall meeting.'
        };
    }
 
})(
    request,
    response
);

Decline Call - 
(function process(
    /*RESTAPIRequest*/ request,
    /*RESTAPIResponse*/ response
) {

    var userId =
        gs.getUserID();


    if (!userId) {

        response.setStatus(401);

        response.setBody({
            success: false,
            code: 'NO_AUTHENTICATED_USER',
            message:
                'Authenticated user could not be identified.'
        });

        return;
    }


    var body =
        request.body.data || {};


    var callSysId =
        (body.call_sys_id || '')
            .toString()
            .trim();


    if (!callSysId) {

        response.setStatus(400);

        response.setBody({
            success: false,
            code: 'CALL_ID_REQUIRED',
            message: 'Call ID is required.'
        });

        return;
    }


    /* ---------------------------------------------------
       GET CALL FIRST
    --------------------------------------------------- */

    var callGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_call'
        );


    if (!callGR.get(callSysId)) {

        response.setStatus(404);

        response.setBody({
            success: false,
            code: 'CALL_NOT_FOUND',
            message: 'Call was not found.'
        });

        return;
    }


    var currentCallState =
        callGR.getValue(
            'u_state'
        ) || '';


    /*
     * ringing:
     * Initial direct call.
     *
     * connected:
     * Participant was invited into an
     * already active conference.
     */
    if (
        currentCallState !== 'ringing' &&
        currentCallState !== 'connected'
    ) {

        response.setStatus(409);

        response.setBody({
            success: false,
            code: 'CALL_NOT_ACTIVE',
            message:
                'This call is no longer available.'
        });

        return;
    }


    var isConferenceDecline =
        currentCallState ===
        'connected';


    /* ---------------------------------------------------
       FIND RINGING PARTICIPANT
    --------------------------------------------------- */

    var participantGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_participant'
        );


    participantGR.addQuery(
        'u_call',
        callSysId
    );

    participantGR.addQuery(
        'u_user',
        userId
    );

    participantGR.addQuery(
        'u_status',
        'ringing'
    );

    participantGR.setLimit(1);

    participantGR.query();


    if (!participantGR.next()) {

        response.setStatus(409);

        response.setBody({
            success: false,
            code: 'NO_RINGING_PARTICIPANT',
            message:
                'No ringing call was found for this user.'
        });

        return;
    }


    /* ---------------------------------------------------
       DECLINE THIS PARTICIPANT
    --------------------------------------------------- */

    var declinedAt =
        new GlideDateTime();


    participantGR.setValue(
        'u_status',
        'declined'
    );

    participantGR.setValue(
        'u_left_at',
        declinedAt
    );

    participantGR.update();


    /* ---------------------------------------------------
       INITIAL DIRECT CALL DECLINE
    --------------------------------------------------- */

    if (!isConferenceDecline) {

        /*
         * The original receiver declined,
         * therefore the entire direct call ends.
         */

        callGR.setValue(
            'u_state',
            'declined'
        );

        callGR.setValue(
            'u_ended_at',
            declinedAt
        );

        callGR.update();


        /*
         * Original caller becomes available.
         */

        var callerId =
            callGR.getValue(
                'u_original_caller'
            );


        if (callerId) {

            var callerPresenceGR =
                new GlideRecord(
                    'x_1806573_servic_0_servicecall_presence'
                );


            callerPresenceGR.addQuery(
                'u_user',
                callerId
            );

            callerPresenceGR.setLimit(1);

            callerPresenceGR.query();


            if (callerPresenceGR.next()) {

                callerPresenceGR.setValue(
                    'u_voice_status',
                    'available'
                );

                callerPresenceGR.setValue(
                    'u_current_call',
                    ''
                );

                callerPresenceGR.update();
            }
        }
    }


    /* ---------------------------------------------------
       DECLINING USER PRESENCE
    --------------------------------------------------- */

    var userPresenceGR =
        new GlideRecord(
            'x_1806573_servic_0_servicecall_presence'
        );


    userPresenceGR.addQuery(
        'u_user',
        userId
    );

    userPresenceGR.setLimit(1);

    userPresenceGR.query();


    if (userPresenceGR.next()) {

        userPresenceGR.setValue(
            'u_voice_status',
            'available'
        );

        userPresenceGR.setValue(
            'u_current_call',
            ''
        );

        userPresenceGR.update();
    }


    /* ---------------------------------------------------
       SUCCESS
    --------------------------------------------------- */

    response.setStatus(200);


    response.setBody({

        success: true,

        code:
            isConferenceDecline
                ? 'CONFERENCE_INVITE_DECLINED'
                : 'CALL_DECLINED',

        message:
            isConferenceDecline
                ? 'Conference invitation declined.'
                : 'Call declined.',

        call_sys_id:
            callSysId,

        state:
            isConferenceDecline
                ? 'connected'
                : 'declined',

        conference_continues:
            isConferenceDecline

    });


})(
    request,
    response
);
    request,
    response
);
