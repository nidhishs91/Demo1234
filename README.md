(function process(request, response) {

    try {

        /* =========================================
           AUTHENTICATED USER
        ========================================= */

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


        /* =========================================
           STORAGE
        ========================================= */

        var meetingIds = {};
        var participantData = {};


        /* =========================================
           1. MEETINGS WHERE USER IS A PARTICIPANT
        ========================================= */

        var participantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting_participant'
            );

        participantGR.addQuery(
            'u_user',
            currentUserSysId
        );

        participantGR.query();


        while (participantGR.next()) {

            var participantMeetingSysId =
                String(
                    participantGR.getValue(
                        'u_meeting'
                    ) || ''
                );


            if (!participantMeetingSysId) {
                continue;
            }


            meetingIds[
                participantMeetingSysId
            ] = true;


            participantData[
                participantMeetingSysId
            ] = {

                participant_sys_id:
                    participantGR.getUniqueValue(),

                role:
                    String(
                        participantGR.getValue(
                            'u_role'
                        ) || ''
                    ),

                invitation_status:
                    String(
                        participantGR.getValue(
                            'u_invitation_status'
                        ) || ''
                    ),

                join_status:
                    String(
                        participantGR.getValue(
                            'u_join_status'
                        ) || ''
                    ),

                joined_at:
                    String(
                        participantGR.getValue(
                            'u_joined_at'
                        ) || ''
                    ),

                left_at:
                    String(
                        participantGR.getValue(
                            'u_left_at'
                        ) || ''
                    ),

                calendar_email:
                    String(
                        participantGR.getValue(
                            'u_calendar_email'
                        ) || ''
                    )
            };
        }


        /* =========================================
           2. MEETINGS WHERE USER IS ORGANIZER
              OR STARTED THE MEETING
        ========================================= */

        var ownedMeetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );


        var ownedQuery =
            ownedMeetingGR.addQuery(
                'u_organizer',
                currentUserSysId
            );


        ownedQuery.addOrCondition(
            'u_started_by',
            currentUserSysId
        );


        ownedMeetingGR.query();


        while (ownedMeetingGR.next()) {

            meetingIds[
                ownedMeetingGR.getUniqueValue()
            ] = true;
        }


        /* =========================================
           NO MEETINGS
        ========================================= */

        var meetingIdArray =
            Object.keys(meetingIds);


        if (meetingIdArray.length === 0) {

            response.setStatus(200);

            return {

                success: true,

                code: 'MY_MEETINGS',

                user_sys_id:
                    currentUserSysId,

                user_name:
                    gs.getUserDisplayName(),

                count: 0,

                meetings: []
            };
        }


        /* =========================================
           3. LOAD ALL MATCHING MEETINGS
        ========================================= */

        var meetings = [];


        var meetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );


        meetingGR.addQuery(
            'sys_id',
            'IN',
            meetingIdArray.join(',')
        );


        meetingGR.orderBy(
            'u_scheduled_start'
        );


        meetingGR.query();


        /* =========================================
           4. BUILD MEETING RESPONSE
        ========================================= */

        while (meetingGR.next()) {

            var meetingSysId =
                String(
                    meetingGR.getUniqueValue()
                );


            var participant =
                participantData[
                    meetingSysId
                ] || {};


            var state =
                String(
                    meetingGR.getValue(
                        'u_state'
                    ) || ''
                );


            var organizerSysId =
                String(
                    meetingGR.getValue(
                        'u_organizer'
                    ) || ''
                );


            var startedBySysId =
                String(
                    meetingGR.getValue(
                        'u_started_by'
                    ) || ''
                );


            var callSysId =
                String(
                    meetingGR.getValue(
                        'u_created_call'
                    ) || ''
                );


            var conferenceId =
                String(
                    meetingGR.getValue(
                        'u_conference_id'
                    ) || ''
                );


            var isOrganizer =
                organizerSysId ===
                currentUserSysId;


            var isStartedByMe =
                startedBySysId ===
                currentUserSysId;


            /*
             * Participant role.
             *
             * If no Meeting Participant record exists
             * but this user is the organizer, return
             * organizer automatically.
             */

            var role =
                String(
                    participant.role || ''
                );


            if (!role && isOrganizer) {
                role = 'organizer';
            }


            /*
             * Organizer is implicitly accepted.
             */

            var invitationStatus =
                String(
                    participant.invitation_status || ''
                );


            if (
                !invitationStatus &&
                isOrganizer
            ) {

                invitationStatus =
                    'accepted';
            }


            var joinStatus =
                String(
                    participant.join_status || ''
                );


            /* =====================================
               ACTION FLAGS FOR DESKTOP
            ===================================== */

            var canStart =
                (
                    state === 'scheduled' &&
                    isOrganizer
                );


            var canJoin =
                (
                    state === 'in progress' &&
                    invitationStatus !== 'declined'
                );


            var canLeave =
                (
                    state === 'in progress' &&
                    joinStatus === 'joined'
                );


            var canEnd =
                (
                    state === 'in progress' &&
                    (
                        isOrganizer ||
                        isStartedByMe
                    )
                );


            /* =====================================
               RESPONSE OBJECT
            ===================================== */

            meetings.push({

                meeting_sys_id:
                    meetingSysId,

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

                description:
                    String(
                        meetingGR.getValue(
                            'u_description'
                        ) || ''
                    ),

                scheduled_start:
                    String(
                        meetingGR.getValue(
                            'u_scheduled_start'
                        ) || ''
                    ),

                scheduled_end:
                    String(
                        meetingGR.getValue(
                            'u_scheduled_end'
                        ) || ''
                    ),

                state:
                    state,

                organizer_sys_id:
                    organizerSysId,

                organizer_name:
                    meetingGR.getDisplayValue(
                        'u_organizer'
                    ),

                started_by_sys_id:
                    startedBySysId,

                started_by_name:
                    startedBySysId
                        ? meetingGR.getDisplayValue(
                            'u_started_by'
                        )
                        : '',

                started_at:
                    String(
                        meetingGR.getValue(
                            'u_started_at'
                        ) || ''
                    ),

                ended_at:
                    String(
                        meetingGR.getValue(
                            'u_ended_at'
                        ) || ''
                    ),

                conference_id:
                    conferenceId,

                call_sys_id:
                    callSysId,


                /* -----------------------------
                   CURRENT USER PARTICIPATION
                ----------------------------- */

                participant_sys_id:
                    String(
                        participant.participant_sys_id ||
                        ''
                    ),

                role:
                    role,

                invitation_status:
                    invitationStatus,

                join_status:
                    joinStatus,

                joined_at:
                    String(
                        participant.joined_at || ''
                    ),

                left_at:
                    String(
                        participant.left_at || ''
                    ),

                calendar_email:
                    String(
                        participant.calendar_email || ''
                    ),


                /* -----------------------------
                   USER RELATIONSHIP
                ----------------------------- */

                is_organizer:
                    isOrganizer,

                is_started_by_me:
                    isStartedByMe,


                /* -----------------------------
                   DESKTOP ACTIONS
                ----------------------------- */

                can_start:
                    canStart,

                can_join:
                    canJoin,

                can_leave:
                    canLeave,

                can_end:
                    canEnd
            });
        }


        /* =========================================
           SUCCESS
        ========================================= */

        response.setStatus(200);

        return {

            success: true,

            code: 'MY_MEETINGS',

            user_sys_id:
                currentUserSysId,

            user_name:
                gs.getUserDisplayName(),

            count:
                meetings.length,

            meetings:
                meetings
        };


    } catch (error) {

        gs.error(
            'ServiceCall My Meetings failed: ' +
            error.message
        );


        response.setStatus(500);


        return {

            success: false,

            code: 'MY_MEETINGS_FAILED',

            message:
                'Unable to retrieve ServiceCall meetings.'
        };
    }

})(
    request,
    response
);
