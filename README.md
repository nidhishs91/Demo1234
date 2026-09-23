            };
        }


        if (
            !scheduledStart ||
            !scheduledEnd
        ) {

            response.setStatus(400);

            return {
                success: false,
                code: 'MEETING_TIME_REQUIRED',
                message: 'Meeting start and end time are required.'
            };
        }


        if (!meetingTimezone) {

            response.setStatus(400);

            return {
                success: false,
                code: 'TIMEZONE_REQUIRED',
                message: 'Meeting time zone is required.'
            };
        }


        /* -----------------------------------------
           LOAD MEETING
        ----------------------------------------- */

        var meetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );


        if (
            !meetingGR.get(
                meetingSysId
            )
        ) {

            response.setStatus(404);

            return {
                success: false,
                code: 'MEETING_NOT_FOUND',
                message: 'Meeting was not found.'
            };
        }


        /* -----------------------------------------
           ORGANIZER SECURITY
        ----------------------------------------- */

        var organizerSysId =
            String(
                meetingGR.getValue(
                    'u_organizer'
                ) || ''
            );


        if (
            organizerSysId !==
            currentUserSysId
        ) {

            response.setStatus(403);

            return {
                success: false,
                code: 'MEETING_EDIT_DENIED',
                message: 'Only the meeting organizer can edit this meeting.'
            };
        }


        /* -----------------------------------------
           STATE VALIDATION
        ----------------------------------------- */

        var meetingState =
            String(
                meetingGR.getValue(
                    'u_state'
                ) || ''
            );


        /*
         * V1 rule:
         *
         * Only meetings that have not started
         * can be edited.
         */
        if (
            meetingState !== 'scheduled'
        ) {

            response.setStatus(409);

            return {
                success: false,
                code: 'MEETING_NOT_EDITABLE',
                message: 'Only scheduled meetings can be edited.'
            };
        }


        /* -----------------------------------------
           PREPARE DATE VALUES

           Electron datetime-local:

           2026-09-19T10:44

           becomes:

           2026-09-19 10:44:00
        ----------------------------------------- */

        var startValue =
            scheduledStart.replace(
                'T',
                ' '
            );


        var endValue =
            scheduledEnd.replace(
                'T',
                ' '
            );


        if (
            startValue.length === 16
        ) {

            startValue += ':00';
        }


        if (
            endValue.length === 16
        ) {

            endValue += ':00';
        }


        /* -----------------------------------------
           DATE / TIME PARSING

           IMPORTANT:
           Same working approach as
           Create Meeting.
        ----------------------------------------- */

        var startGdt =
            new GlideDateTime();


        startGdt.setDisplayValue(
            startValue
        );


        var endGdt =
            new GlideDateTime();


        endGdt.setDisplayValue(
            endValue
        );


        /*
         * Temporary diagnostic logging.
         */
        gs.info(
            '[ServiceCall Update Meeting Time Test]' +
            ' | Received Start=' +
            scheduledStart +
            ' | Received End=' +
            scheduledEnd +
            ' | Received Timezone=' +
            meetingTimezone +
            ' | Start Internal=' +
            startGdt.getValue() +
            ' | Start Display=' +
            startGdt.getDisplayValue() +
            ' | End Internal=' +
            endGdt.getValue() +
            ' | End Display=' +
            endGdt.getDisplayValue()
        );


        /* -----------------------------------------
           DATE VALIDATION
        ----------------------------------------- */

        if (
            !startGdt.isValid() ||
            !endGdt.isValid()
        ) {

            response.setStatus(400);

            return {
                success: false,
                code: 'INVALID_MEETING_TIME',
                message: 'Meeting start or end time is invalid.'
            };
        }


        if (
            endGdt.getNumericValue() <=
            startGdt.getNumericValue()
        ) {

            response.setStatus(400);

            return {
                success: false,
                code: 'INVALID_MEETING_RANGE',
                message: 'Meeting end time must be after the start time.'
            };
        }


        /* -----------------------------------------
           VALIDATE PARTICIPANTS
        ----------------------------------------- */

        var validParticipants = [];

        var seenUsers = {};


        /*
         * Organizer must never be added
         * again as an attendee.
         */
        seenUsers[
            organizerSysId
        ] = true;


        for (
            var i = 0; i < participants.length; i++
        ) {

            var userSysId =
                String(
                    participants[i] || ''
                ).trim();


            if (!userSysId) {
                continue;
            }


            /*
             * Ignore duplicates and organizer.
             */
            if (
                seenUsers[
                    userSysId
                ]
            ) {

                continue;
            }


            var userGR =
                new GlideRecord(
                    'sys_user'
                );


            if (
                !userGR.get(
                    userSysId
                )
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code: 'INVALID_PARTICIPANT',
                    message: 'Participant user was not found.',
                    participant_sys_id: userSysId
                };
            }


            /*
             * ServiceNow Boolean fields may
             * return "1" or "true".
             */
            var activeValue =
                String(
                    userGR.getValue(
                        'active'
                    )
                );


            if (
                activeValue !== '1' &&
                activeValue !== 'true'
            ) {

                response.setStatus(400);

                return {
                    success: false,
                    code: 'INACTIVE_PARTICIPANT',
                    message: 'Selected participant is inactive.',
                    participant_sys_id: userSysId,
                    participant_name: userGR.getDisplayValue()
                };
            }


            seenUsers[
                userSysId
            ] = true;


            validParticipants.push(
                userSysId
            );
        }


        /*
         * At least one attendee is required.
         */
        if (
            validParticipants.length === 0
        ) {

            response.setStatus(400);

            return {
                success: false,
                code: 'PARTICIPANT_REQUIRED',
                message: 'Select at least one active participant.'
            };
        }


        /* -----------------------------------------
           UPDATE MEETING RECORD
        ----------------------------------------- */

        meetingGR.setValue(
            'u_title',
            title
        );


        meetingGR.setValue(
            'u_description',
            description
        );


        meetingGR.setValue(
            'u_scheduled_start',
            startGdt
        );


        meetingGR.setValue(
            'u_scheduled_end',
            endGdt
        );

        /*
         * Any successful edit counts as
         * meeting activity.
         *
         * This includes:
         * - title
         * - description
         * - start/end time
         * - participant changes
         */
        meetingGR.setValue(
            'u_last_activity_at',
            new GlideDateTime()
        );

        var updatedMeetingSysId =
            meetingGR.update();


        if (!updatedMeetingSysId) {

            throw new Error(
                'Unable to update meeting record.'
            );
        }


        /* -----------------------------------------
           BUILD REQUESTED ATTENDEE MAP
        ----------------------------------------- */

        var requestedAttendees = {};


        for (
            var r = 0; r < validParticipants.length; r++
        ) {

            requestedAttendees[
                validParticipants[r]
            ] = true;
        }


        /* -----------------------------------------
           EXISTING PARTICIPANTS
        ----------------------------------------- */

        var existingAttendees = {};


        var participantGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting_participant'
            );


        participantGR.addQuery(
            'u_meeting',
            meetingSysId
        );


        participantGR.query();


        while (
            participantGR.next()
        ) {

            var existingUserSysId =
                String(
                    participantGR.getValue(
                        'u_user'
                    ) || ''
                );


            var existingRole =
                String(
                    participantGR.getValue(
                        'u_role'
                    ) || ''
                );


            /*
             * NEVER modify/delete organizer
             * participant record here.
             */
            if (
                existingRole ===
                'organizer' ||
                existingUserSysId ===
                organizerSysId
            ) {

                continue;
            }


            /*
             * Attendee still exists in the
             * updated participant list.
             *
             * Preserve their existing
             * invitation status.
             */
            if (
                requestedAttendees[
                    existingUserSysId
                ]
            ) {

                existingAttendees[
                    existingUserSysId
                ] = true;

                continue;
            }


            /*
             * This attendee was removed
             * from the meeting.
             */
            participantGR.deleteRecord();
        }


        /* -----------------------------------------
           ADD NEW ATTENDEES
        ----------------------------------------- */

        for (
            var p = 0; p < validParticipants.length; p++
        ) {

            var attendeeSysId =
                validParticipants[p];


            /*
             * Already exists.
             */
            if (
                existingAttendees[
                    attendeeSysId
                ]
            ) {

                continue;
            }


            var newParticipantGR =
                new GlideRecord(
                    'x_1806573_servic_0_servicecall_meeting_participant'
                );


            newParticipantGR.initialize();


            newParticipantGR.setValue(
                'u_meeting',
                meetingSysId
            );


            newParticipantGR.setValue(
                'u_user',
                attendeeSysId
            );


            newParticipantGR.setValue(
                'u_role',
                'attendee'
            );


            /*
             * New attendee receives a fresh
             * invitation state.
             */
            newParticipantGR.setValue(
                'u_invitation_status',
                'invited'
            );


            newParticipantGR.setValue(
                'u_join_status',
                'not joined'
            );


            var newParticipantSysId =
                newParticipantGR.insert();


            if (!newParticipantSysId) {

                throw new Error(
                    'Unable to add meeting participant.'
                );
            }
        }

        /* -----------------------------------------
           RELOAD UPDATED MEETING
        ----------------------------------------- */

        var updatedMeetingGR =
            new GlideRecord(
                'x_1806573_servic_0_servicecall_meeting'
            );


        updatedMeetingGR.get(
            meetingSysId
        );


        /* -----------------------------------------
           SUCCESS RESPONSE
        ----------------------------------------- */

        response.setStatus(200);


        return {

            success: true,

            code: 'MEETING_UPDATED',

            message: 'Meeting updated successfully.',

            meeting_sys_id: meetingSysId,

            meeting_number: updatedMeetingGR.getDisplayValue(
                'number'
            ) || '',

            title: updatedMeetingGR.getValue(
                'u_title'
            ) || '',

            organizer_sys_id: organizerSysId,

            scheduled_start: updatedMeetingGR.getValue(
                'u_scheduled_start'
            ),

            scheduled_end: updatedMeetingGR.getValue(
                'u_scheduled_end'
            ),

            timezone: meetingTimezone,

            state: updatedMeetingGR.getValue(
                'u_state'
            ),

            invited_participants: validParticipants.length
        };


    } catch (error) {

        gs.error(
            'ServiceCall Update Meeting failed: ' +
            error.message
        );


        response.setStatus(500);


        return {
            success: false,
            code: 'UPDATE_MEETING_FAILED',
            message: 'Unable to update the ServiceCall meeting.'
        };
    }

})(
    request,
    response
);


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
                message:
                    'Meeting sys_id is required.'
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
                message:
                    'Meeting was not found.'
            };
        }


        var meetingState =
            String(
                meetingGR.getValue(
                    'u_state'
                ) || ''
            );


        /* -----------------------------------------
           ALREADY CANCELLED
        ----------------------------------------- */

        if (
            meetingState ===
            'cancelled'
        ) {

            response.setStatus(200);

            return {
                success: true,
                code:
                    'MEETING_ALREADY_CANCELLED',

                meeting_sys_id:
                    meetingGR.getUniqueValue(),

                meeting_number:
                    meetingGR.getDisplayValue(
                        'number'
                    ),

                state:
                    'cancelled'
            };
        }


        /* -----------------------------------------
           ONLY SCHEDULED MEETINGS CAN BE CANCELLED
        ----------------------------------------- */

        if (
            meetingState !==
            'scheduled'
        ) {

            response.setStatus(409);

            return {
                success: false,
                code:
                    'MEETING_CANNOT_BE_CANCELLED',

                message:
                    meetingState ===
                    'in progress'
                        ? 'This meeting has already started. End the meeting instead.'
                        : 'This meeting cannot be cancelled.'
            };
        }


        /* -----------------------------------------
           ONLY ORGANIZER CAN CANCEL
        ----------------------------------------- */

        var organizerSysId =
            String(
                meetingGR.getValue(
                    'u_organizer'
                ) || ''
            );


        if (
            currentUserSysId !==
            organizerSysId
        ) {

            response.setStatus(403);

            return {
                success: false,
                code:
                    'NOT_AUTHORIZED_TO_CANCEL_MEETING',

                message:
                    'Only the meeting organizer can cancel this meeting.'
            };
        }


        /* -----------------------------------------
           CANCEL MEETING
        ----------------------------------------- */

        meetingGR.setValue(
            'u_state',
            'cancelled'
        );

        meetingGR.update();


        /* -----------------------------------------
           RESPONSE
        ----------------------------------------- */

        response.setStatus(200);

        return {

            success: true,

            code:
                'MEETING_CANCELLED',

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

            state:
                String(
                    meetingGR.getValue(
                        'u_state'
                    ) || ''
                ),

            organizer:
                organizerSysId,

            organizer_name:
                meetingGR.getDisplayValue(
                    'u_organizer'
                )
        };


    } catch (error) {

        gs.error(
            'ServiceCall Cancel Meeting failed: ' +
            error.message
        );

        response.setStatus(500);

        return {
            success: false,
            code:
                'MEETING_CANCEL_FAILED',
            message:
                'Unable to cancel the ServiceCall meeting.'
        };
    }

})(
    request,
    response
);
