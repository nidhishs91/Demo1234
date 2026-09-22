function renderParticipants(
    participants
) {

    latestCallParticipants =
        Array.isArray(participants)
            ? participants
            : [];


    /* -------------------------
       COUNT
    ------------------------- */

    if (participantsPanelCount) {

        participantsPanelCount
            .textContent =
                String(
                    latestCallParticipants
                        .length
                );
    }


    if (!participantsPanelBody) {
        return;
    }


    participantsPanelBody.innerHTML =
        '';


    /* -------------------------
       SECTION LABEL
    ------------------------- */

    const sectionLabel =
        document.createElement(
            'div'
        );

    sectionLabel.className =
        'participants-section-label';

    sectionLabel.textContent =
        isMeeting
            ? 'In this meeting'
            : 'In this call';


    participantsPanelBody
        .appendChild(
            sectionLabel
        );


    /* -------------------------
       EMPTY STATE
    ------------------------- */

    if (
        latestCallParticipants.length === 0
    ) {

        const empty =
            document.createElement(
                'div'
            );

        empty.className =
            'participants-empty';

        empty.textContent =
            'No active participants.';


        participantsPanelBody
            .appendChild(
                empty
            );

        return;
    }


    /* -------------------------
       PARTICIPANTS
    ------------------------- */

    latestCallParticipants.forEach(
        participant => {

            const row =
                document.createElement(
                    'div'
                );

            row.className =
                'live-participant';


            /* -------------------------
               AVATAR
            ------------------------- */

            const participantAvatar =
                document.createElement(
                    'div'
                );

            participantAvatar.className =
                'live-participant-avatar';

            participantAvatar.textContent =
                getParticipantInitials(
                    participant.name
                );


            /* -------------------------
               INFO
            ------------------------- */

            const info =
                document.createElement(
                    'div'
                );

            info.className =
                'live-participant-info';


            const name =
                document.createElement(
                    'div'
                );

            name.className =
                'live-participant-name';

            name.textContent =
                participant.name ||
                'Unknown User';


            const meta =
                document.createElement(
                    'div'
                );

            meta.className =
                'live-participant-meta';


            const metaParts =
                [];


            if (
                participant.is_owner === true
            ) {

                metaParts.push(
                    isMeeting
                        ? 'Organizer'
                        : 'Owner'
                );

            } else if (
                participant.role
            ) {

                metaParts.push(
                    participant.role
                );
            }


            if (
                participant.department
            ) {

                metaParts.push(
                    participant.department
                );
            }


            meta.textContent =
                metaParts.join(
                    ' • '
                ) ||
                (
                    isMeeting
                        ? 'Meeting participant'
                        : 'Call participant'
                );


            info.appendChild(
                name
            );

            info.appendChild(
                meta
            );


            /* -------------------------
               STATUS
            ------------------------- */

            const participantStatus =
                document.createElement(
                    'div'
                );

            participantStatus.className =
                'live-participant-status';


            const statusDot =
                document.createElement(
                    'span'
                );

            statusDot.className =
                'live-participant-status-dot';


            const statusLabel =
                document.createElement(
                    'span'
                );

            statusLabel.textContent =
                getParticipantStatusLabel(
                    participant.status
                );


            participantStatus.appendChild(
                statusDot
            );

            participantStatus.appendChild(
                statusLabel
            );


            /* -------------------------
               BUILD ROW
            ------------------------- */

            row.appendChild(
                participantAvatar
            );

            row.appendChild(
                info
            );

            row.appendChild(
                participantStatus
            );


            participantsPanelBody
                .appendChild(
                    row
                );
        }
    );
}
