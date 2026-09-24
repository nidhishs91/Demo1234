<!DOCTYPE html>
<html lang="en">

<head>

    <meta charset="UTF-8">

    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>
        ServiceCall Desktop
    </title>


    <style>
        * {
            box-sizing: border-box;
        }


        body {
            margin: 0;

            font-family:
                Inter,
                "Segoe UI",
                Arial,
                sans-serif;

            background: #f5f7f7;
            color: #182825;

            overflow: hidden;
        }


        button,
        input {
            font: inherit;
        }


        .app {
            display: flex;

            height: 100vh;
            width: 100vw;
        }


        /* =================================================
           SIDEBAR
        ================================================= */

        .sidebar {
            width: 220px;
            min-width: 220px;

            background: #102f2b;
            color: white;

            display: flex;
            flex-direction: column;

            padding: 22px 14px;
        }


        .brand {
            display: flex;
            align-items: center;

            gap: 11px;

            padding: 0 10px 28px;
        }


        .brand-mark {
            width: 34px;
            height: 34px;

            border-radius: 10px;

            background: #55d6a9;
            color: #10332c;

            display: flex;
            align-items: center;
            justify-content: center;

            font-weight: 800;
            font-size: 17px;
        }


        .brand-name {
            font-size: 18px;
            font-weight: 700;
        }


        .navigation {
            display: flex;
            flex-direction: column;

            gap: 5px;
        }


        .nav-button {
            width: 100%;

            border: 0;

            background: transparent;
            color: #cfe0dc;

            text-align: left;

            padding: 11px 13px;

            border-radius: 9px;

            cursor: pointer;

            transition:
                background 0.15s ease,
                color 0.15s ease;
        }


        .nav-button:hover {
            background:
                rgba(255,
                    255,
                    255,
                    0.07);

            color: white;
        }


        .nav-button.active {
            background: #1d4d44;

            color: white;

            font-weight: 600;
        }


        .sidebar-bottom {
            margin-top: auto;
        }


        /* =================================================
           MAIN AREA
        ================================================= */

        .main {
            flex: 1;

            min-width: 0;

            display: flex;
            flex-direction: column;
        }


        .topbar {
            height: 68px;
            min-height: 68px;

            background: white;

            border-bottom:
                1px solid #e4e9e7;

            display: flex;
            align-items: center;
            justify-content: space-between;

            padding: 0 30px;
        }


        .topbar-title {
            font-size: 18px;
            font-weight: 650;
        }


        .connection-pill {
            padding: 7px 12px;

            border-radius: 999px;

            background: #edf3f1;
            color: #49635d;

            font-size: 12px;
            font-weight: 600;
        }


        .connection-pill.connected {
            background: #e3f7ef;
            color: #147357;
        }

        /* =================================================
   TOP BAR PRESENCE
================================================= */

        .topbar-right {
            display: flex;
            align-items: center;

            gap: 12px;
        }


        /* -------------------------------------------------
   PRESENCE CONTROL
------------------------------------------------- */

        .presence-control {
            position: relative;
        }


        .presence-button {
            height: 36px;

            display: flex;
            align-items: center;

            gap: 8px;

            padding: 0 12px;

            border:
                1px solid #d8e2df;

            border-radius: 999px;

            background: white;
            color: #29463f;

            font-size: 12px;
            font-weight: 650;

            cursor: pointer;

            transition:
                background 0.15s ease,
                border-color 0.15s ease;
        }


        .presence-button:hover {
            background: #f5f9f7;

            border-color: #bccdc8;
        }


        .presence-dot,
        .presence-option-dot {
            width: 9px;
            height: 9px;

            flex-shrink: 0;

            border-radius: 50%;
        }


        .presence-dot.available,
        .presence-option-dot.available {
            background: #22a06b;
        }


        .presence-dot.busy,
        .presence-option-dot.busy {
            background: #d94c4c;
        }


        .presence-dot.away,
        .presence-option-dot.away {
            background: #e0a21a;
        }


        .presence-dot.out-of-office,
        .presence-option-dot.out-of-office {
            background: #7c63c7;
        }


        .presence-dot.in-call {
            background: #d94c4c;
        }


        .presence-dot.offline,
        .presence-option-dot.offline {
            background: #000000;
        }


        .presence-chevron {
            margin-left: 2px;

            color: #71827d;

            font-size: 10px;
        }


        /* -------------------------------------------------
   PRESENCE MENU
------------------------------------------------- */

        .presence-menu {
            position: absolute;

            z-index: 2000;

            top: calc(100% + 8px);
            right: 0;

            width: 260px;

            padding: 7px;

            background: white;

            border:
                1px solid #dce5e2;

            border-radius: 12px;

            box-shadow:
                0 14px 35px rgba(16, 47, 43, 0.15);
        }


        .presence-option {
            width: 100%;

            display: flex;
            align-items: center;

            gap: 10px;

            padding: 10px 11px;

            border: 0;

            border-radius: 8px;

            background: transparent;
            color: #29463f;

            text-align: left;

            font-size: 13px;
            font-weight: 600;

            cursor: pointer;
        }


        .presence-option:hover {
            background: #f0f6f4;
        }


        /* -------------------------------------------------
   OUT OF OFFICE REASON
------------------------------------------------- */

        .oof-reason-panel {
            margin-top: 7px;

            padding: 12px;

            border-top:
                1px solid #e7ecea;
        }


        .oof-reason-label {
            display: block;

            margin-bottom: 7px;

            color: #405b54;

            font-size: 12px;
            font-weight: 650;
        }


        .oof-reason-input {
            width: 100%;

            min-height: 72px;

            padding: 9px 10px;

            border:
                1px solid #d3dedb;

            border-radius: 8px;

            outline: none;

            resize: vertical;

            font-family: inherit;
            font-size: 12px;

            color: #29463f;
        }


        .oof-reason-input:focus {
            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61, 145, 124, 0.10);
        }


        .oof-reason-actions {
            display: flex;

            justify-content: flex-end;

            gap: 7px;

            margin-top: 9px;
        }


        .oof-reason-cancel,
        .oof-reason-save {
            padding: 7px 11px;

            border-radius: 7px;

            font-size: 11px;
            font-weight: 650;

            cursor: pointer;
        }


        .oof-reason-cancel {
            border:
                1px solid #d1dcda;

            background: white;
            color: #536963;
        }


        .oof-reason-save {
            border:
                1px solid #126e5a;

            background: #126e5a;
            color: white;
        }


        .oof-reason-save:hover {
            background: #0e5e4d;
        }

        /* -------------------------------------------------
   RESET PRESENCE
------------------------------------------------- */

        .presence-reset-divider {
            height: 1px;

            margin: 7px 4px;

            background: #e7ecea;
        }


        .presence-reset-button {
            width: 100%;

            display: flex;
            align-items: center;

            gap: 10px;

            padding: 10px 11px;

            border: 0;

            border-radius: 8px;

            background: transparent;
            color: #526963;

            text-align: left;

            font-size: 13px;
            font-weight: 600;

            cursor: pointer;

            transition:
                background 0.15s ease,
                color 0.15s ease;
        }


        .presence-reset-button:hover {
            background: #f0f6f4;

            color: #185d4d;
        }


        .presence-reset-icon {
            width: 9px;

            display: inline-flex;
            align-items: center;
            justify-content: center;

            color: #71827d;

            font-size: 16px;
            font-weight: 700;
        }

        .content {
            flex: 1;

            overflow-y: auto;

            padding: 30px;
        }


        .view {
            display: none;
        }


        .view.active {
            display: block;
        }


        /* =================================================
           GENERIC
        ================================================= */

        .page-heading {
            display: flex;

            justify-content: space-between;
            align-items: flex-start;

            gap: 20px;

            margin-bottom: 26px;
        }


        .page-heading h1 {
            margin: 0 0 7px;

            font-size: 28px;
        }


        .page-heading p {
            margin: 0;

            color: #687a76;

            font-size: 14px;
        }


        .primary-button {
            border: 0;

            background: #126e5a;
            color: white;

            padding: 11px 17px;

            border-radius: 8px;

            font-weight: 600;

            cursor: pointer;

            transition:
                background 0.15s ease,
                transform 0.15s ease;
        }


        .primary-button:hover {
            background: #0e5e4d;
        }


        .secondary-button {
            border:
                1px solid #cfdad7;

            background: white;
            color: #25463f;

            padding: 10px 16px;

            border-radius: 8px;

            font-weight: 600;

            cursor: pointer;

            transition:
                background 0.15s ease,
                border-color 0.15s ease;
        }


        .secondary-button:hover {
            background: #f6f9f8;
            border-color: #b9cac5;
        }


        .card {
            background: white;

            border:
                1px solid #e1e7e5;

            border-radius: 13px;

            padding: 22px;

            box-shadow:
                0 2px 8px rgba(23,
                    51,
                    45,
                    0.035);
        }


        /* =================================================
           HOME
        ================================================= */

        .welcome-card {
            background:
                linear-gradient(120deg,
                    #143f37,
                    #1a6958);

            color: white;

            border-radius: 16px;

            padding: 30px;
        }


        .welcome-card h1 {
            margin: 0 0 10px;
        }


        .welcome-card p {
            margin: 0;

            color: #d4ebe5;

            max-width: 600px;

            line-height: 1.6;
        }


        .home-actions {
            margin-top: 22px;

            display: flex;

            gap: 10px;
        }


        /* =================================================
           MEETINGS
        ================================================= */

        .meeting-section-title {
            margin: 27px 0 13px;

            font-size: 15px;
            font-weight: 700;

            color: #344c47;
        }


        /* -------------------------------------------------
           MEETING TOOLBAR
        ------------------------------------------------- */

        .meetings-toolbar {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 16px;

            margin-bottom: 18px;
        }


        /*
         * Search container.
         *
         * Keeping the icon inside the same rounded
         * surface makes the search feel more like
         * a desktop application control.
         */

        .meeting-search-wrapper {
            position: relative;

            width: 100%;
            max-width: 440px;
        }


        .meeting-search-icon {
            position: absolute;

            left: 15px;
            top: 50%;

            transform:
                translateY(-50%);

            width: 18px;
            height: 18px;

            color: #758782;

            pointer-events: none;

            transition:
                color 0.18s ease;
        }


        .meeting-search {
            width: 100%;

            height: 44px;

            padding:
                0 42px 0 44px;

            border:
                1px solid #d6e0dd;

            border-radius: 11px;

            outline: none;

            background: white;

            color: #1c332e;

            font-size: 14px;

            box-shadow:
                0 1px 3px rgba(16,
                    47,
                    43,
                    0.025);

            transition:
                border-color 0.18s ease,
                box-shadow 0.18s ease,
                background 0.18s ease;
        }


        .meeting-search::placeholder {
            color: #91a09c;
        }


        .meeting-search:hover {
            border-color: #bccdc8;
        }


        .meeting-search:focus {
            background: #ffffff;

            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61,
                    145,
                    124,
                    0.11);
        }


        .meeting-search-wrapper:focus-within .meeting-search-icon {
            color: #25745f;
        }


        /*
         * Native search inputs can show their own
         * clear control. We remove that because
         * renderer.js can later provide one
         * consistently across platforms.
         */

        .meeting-search::-webkit-search-cancel-button {
            display: none;
        }


        .meeting-search-clear {
            position: absolute;

            right: 10px;
            top: 50%;

            transform:
                translateY(-50%);

            width: 27px;
            height: 27px;

            border: 0;

            border-radius: 7px;

            background: transparent;

            color: #71817d;

            display: none;

            align-items: center;
            justify-content: center;

            font-size: 18px;
            line-height: 1;

            cursor: pointer;

            transition:
                background 0.15s ease,
                color 0.15s ease;
        }


        .meeting-search-clear:hover {
            background: #edf3f1;

            color: #244a41;
        }

        /* -------------------------------------------------
   MEETING STATUS FILTER
------------------------------------------------- */

        .meeting-status-filter-wrapper {
            display: flex;

            align-items: center;

            gap: 9px;

            flex-shrink: 0;
        }


        .meeting-status-filter-label {
            color: #60736e;

            font-size: 13px;
            font-weight: 650;
        }


        .meeting-status-filter {
            height: 44px;

            min-width: 150px;

            padding:
                0 38px 0 13px;

            border:
                1px solid #d6e0dd;

            border-radius: 11px;

            outline: none;

            background: white;

            color: #29463f;

            font-size: 14px;
            font-weight: 600;

            cursor: pointer;

            box-shadow:
                0 1px 3px rgba(16,
                    47,
                    43,
                    0.025);

            transition:
                border-color 0.18s ease,
                box-shadow 0.18s ease,
                background 0.18s ease;
        }


        .meeting-status-filter:hover {
            border-color: #bccdc8;
        }


        .meeting-status-filter:focus {
            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61,
                    145,
                    124,
                    0.11);
        }

        /* -------------------------------------------------
           MEETING LIST
        ------------------------------------------------- */

        .meetings-list {
            display: grid;

            gap: 12px;
        }


        .meeting-card {
            background: white;

            border:
                1px solid #e0e7e5;

            border-radius: 12px;

            padding: 18px 20px;

            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 20px;

            transition:
                border-color 0.15s ease,
                box-shadow 0.15s ease,
                transform 0.15s ease;
        }


        .meeting-card:hover {
            border-color: #cedbd7;

            box-shadow:
                0 3px 12px rgba(23,
                    51,
                    45,
                    0.045);
        }


        .meeting-card-left {
            min-width: 0;
        }


        .meeting-title {
            font-size: 16px;
            font-weight: 650;

            margin-bottom: 6px;
        }


        .meeting-meta {
            color: #71807d;

            font-size: 13px;

            line-height: 1.6;
        }


        .meeting-actions {
            display: flex;

            align-items: center;

            gap: 8px;

            flex-shrink: 0;
        }


        .meeting-status {
            display: inline-flex;

            align-items: center;

            padding: 5px 9px;

            border-radius: 999px;

            background: #edf1f0;
            color: #61716d;

            font-size: 11px;
            font-weight: 700;

            text-transform: capitalize;
        }


        .meeting-status.in-progress {
            background: #e2f7ee;
            color: #137254;
        }


        .meeting-status.scheduled {
            background: #e8f1fb;
            color: #326a9e;
        }


        .empty-state {
            background: white;

            border:
                1px dashed #ccd8d4;

            border-radius: 12px;

            padding: 35px;

            text-align: center;

            color: #75837f;
        }


        .loading {
            padding: 25px;

            text-align: center;

            color: #71807d;
        }


        /* -------------------------------------------------
           MEETING PAGINATION
        ------------------------------------------------- */

        .meeting-pagination {
            display: flex;

            align-items: center;
            justify-content: center;

            flex-wrap: wrap;

            gap: 7px;

            min-height: 38px;

            margin-top: 20px;

            padding-bottom: 10px;
        }


        .meeting-page-button {
            min-width: 36px;
            height: 36px;

            padding: 0 10px;

            border:
                1px solid #d2ddda;

            border-radius: 9px;

            background: white;
            color: #38534d;

            font-size: 13px;
            font-weight: 650;

            cursor: pointer;

            transition:
                background 0.15s ease,
                border-color 0.15s ease,
                color 0.15s ease,
                transform 0.15s ease;
        }


        .meeting-page-button:hover:not(:disabled) {
            background: #eef5f2;

            border-color: #b8cbc5;

            color: #185d4d;
        }


        .meeting-page-button.active {
            background: #126e5a;

            border-color: #126e5a;

            color: white;
        }


        .meeting-page-button:disabled {
            opacity: 0.42;

            cursor: default;
        }


        .meeting-page-info {
            color: #71807d;

            font-size: 12px;

            margin: 0 5px;
        }


        /* =================================================
   MEETING DETAILS MODAL
================================================= */

        .meeting-details-modal {
            position: fixed;

            inset: 0;

            z-index: 1000;

            display: none;

            align-items: center;
            justify-content: center;

            padding: 28px;
        }


        .meeting-details-modal.open {
            display: flex;
        }


        .meeting-details-backdrop {
            position: absolute;

            inset: 0;

            background:
                rgba(10,
                    28,
                    24,
                    0.42);

            backdrop-filter:
                blur(2px);
        }


        .meeting-details-dialog {
            position: relative;

            width: min(680px,
                    100%);

            max-height:
                calc(100vh - 56px);

            background: white;

            border:
                1px solid #dce5e2;

            border-radius: 16px;

            box-shadow:
                0 24px 70px rgba(10,
                    37,
                    31,
                    0.20);

            display: flex;
            flex-direction: column;

            overflow: hidden;
        }


        .meeting-details-header {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 20px;

            padding: 20px 22px;

            border-bottom:
                1px solid #e7ecea;
        }


        .meeting-details-heading {
            font-size: 17px;
            font-weight: 700;

            color: #193a32;
        }


        .meeting-details-number {
            margin-top: 3px;

            color: #7b8c87;

            font-size: 12px;
        }


        .meeting-details-close {
            width: 34px;
            height: 34px;

            border: 0;

            border-radius: 9px;

            background: transparent;

            color: #667b75;

            font-size: 22px;
            line-height: 1;

            cursor: pointer;
        }


        .meeting-details-close:hover {
            background: #edf3f1;

            color: #234a40;
        }


        .meeting-details-content {
            overflow-y: auto;

            padding: 22px;
        }


        .meeting-details-title-row {
            display: flex;

            align-items: flex-start;
            justify-content: space-between;

            gap: 18px;

            margin-bottom: 22px;
        }


        .meeting-details-title {
            margin: 0;

            color: #193a32;

            font-size: 22px;
            line-height: 1.3;
        }


        .meeting-details-section {
            padding: 19px 0;

            border-top:
                1px solid #edf1f0;
        }


        .meeting-details-section-title {
            margin-bottom: 14px;

            color: #294a42;

            font-size: 14px;
            font-weight: 700;
        }


        .meeting-details-label {
            margin-bottom: 5px;

            color: #7a8985;

            font-size: 11px;
            font-weight: 700;

            text-transform: uppercase;

            letter-spacing: 0.035em;
        }


        .meeting-details-value,
        .meeting-details-description {
            color: #2d4540;

            font-size: 14px;

            line-height: 1.55;

            word-break: break-word;
        }


        .meeting-details-description {
            white-space: pre-wrap;
        }


        .meeting-details-grid {
            display: grid;

            grid-template-columns:
                repeat(2,
                    minmax(0,
                        1fr));

            gap: 18px 28px;
        }


        .meeting-details-field {
            min-width: 0;
        }


        .meeting-details-participants-header {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 12px;
        }


        .meeting-details-count {
            min-width: 27px;
            height: 27px;

            padding: 0 8px;

            border-radius: 999px;

            background: #edf4f2;

            color: #376158;

            display: flex;
            align-items: center;
            justify-content: center;

            font-size: 12px;
            font-weight: 700;
        }


        .meeting-details-participants {
            display: grid;

            gap: 9px;
        }


        .meeting-details-participant {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 16px;

            padding: 12px 13px;

            border:
                1px solid #e3e9e7;

            border-radius: 10px;

            background: #fafcfb;
        }


        .meeting-details-participant-main {
            min-width: 0;
        }


        .meeting-details-participant-name {
            color: #263f39;

            font-size: 13px;
            font-weight: 650;

            overflow: hidden;

            text-overflow: ellipsis;

            white-space: nowrap;
        }


        .meeting-details-participant-role {
            margin-top: 3px;

            color: #7b8b87;

            font-size: 11px;

            text-transform: capitalize;
        }


        .meeting-details-participant-statuses {
            display: flex;

            align-items: center;
            justify-content: flex-end;

            flex-wrap: wrap;

            gap: 6px;
        }


        .meeting-details-badge {
            padding: 4px 8px;

            border-radius: 999px;

            background: #edf2f1;

            color: #5e716c;

            font-size: 10px;
            font-weight: 700;

            text-transform: capitalize;
        }


        .meeting-details-empty {
            padding: 18px;

            border:
                1px dashed #d2dcda;

            border-radius: 10px;

            color: #7b8b87;

            text-align: center;

            font-size: 13px;
        }


        .meeting-details-footer {
            display: flex;

            justify-content: flex-end;

            padding: 15px 22px;

            border-top:
                1px solid #e7ecea;

            background: #fbfcfc;
        }

        /* =================================================
   SCHEDULE MEETING MODAL
================================================= */

        .schedule-meeting-timezone {
            display: flex;
            align-items: center;

            gap: 5px;

            margin-top: -7px;
            margin-bottom: 19px;

            color: #71827d;

            font-size: 12px;
        }


        .schedule-meeting-timezone-label {
            font-weight: 650;

            color: #526b65;
        }

        .schedule-meeting-modal {
            position: fixed;

            inset: 0;

            z-index: 1010;

            display: none;

            align-items: center;
            justify-content: center;

            padding: 28px;
        }


        .schedule-meeting-modal.open {
            display: flex;
        }


        .schedule-meeting-backdrop {
            position: absolute;

            inset: 0;

            background:
                rgba(10,
                    28,
                    24,
                    0.42);

            backdrop-filter:
                blur(2px);
        }


        .schedule-meeting-dialog {
            position: relative;

            width: min(700px,
                    100%);

            max-height:
                calc(100vh - 56px);

            background: white;

            border:
                1px solid #dce5e2;

            border-radius: 16px;

            box-shadow:
                0 24px 70px rgba(10,
                    37,
                    31,
                    0.20);

            display: flex;
            flex-direction: column;

            overflow: hidden;
        }


        .schedule-meeting-header {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 20px;

            padding: 20px 22px;

            border-bottom:
                1px solid #e7ecea;
        }


        .schedule-meeting-heading {
            color: #193a32;

            font-size: 18px;
            font-weight: 700;
        }


        .schedule-meeting-subtitle {
            margin-top: 4px;

            color: #7b8c87;

            font-size: 12px;
        }


        .schedule-meeting-close {
            width: 34px;
            height: 34px;

            border: 0;

            border-radius: 9px;

            background: transparent;

            color: #667b75;

            font-size: 22px;
            line-height: 1;

            cursor: pointer;
        }


        .schedule-meeting-close:hover {
            background: #edf3f1;

            color: #234a40;
        }


        .schedule-meeting-content {
            padding: 22px;

            overflow-y: auto;
        }


        .schedule-meeting-field {
            margin-bottom: 19px;
        }


        .schedule-meeting-label {
            display: block;

            margin-bottom: 7px;

            color: #38544d;

            font-size: 13px;
            font-weight: 650;
        }


        .schedule-meeting-required {
            color: #b34747;
        }


        .schedule-meeting-input,
        .schedule-meeting-textarea {
            width: 100%;

            box-sizing: border-box;

            border:
                1px solid #d6e0dd;

            border-radius: 10px;

            outline: none;

            background: white;

            color: #29463f;

            font-family: inherit;

            font-size: 14px;

            transition:
                border-color 0.18s ease,
                box-shadow 0.18s ease;
        }


        .schedule-meeting-input {
            height: 44px;

            padding: 0 13px;
        }


        .schedule-meeting-textarea {
            min-height: 96px;

            padding: 11px 13px;

            resize: vertical;

            line-height: 1.5;
        }


        .schedule-meeting-input:focus,
        .schedule-meeting-textarea:focus {
            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61,
                    145,
                    124,
                    0.11);
        }


        .schedule-meeting-time-grid {
            display: grid;

            grid-template-columns:
                repeat(2,
                    minmax(0,
                        1fr));

            gap: 16px;
        }


        .schedule-meeting-people-search-wrapper {
            position: relative;
        }


        .schedule-meeting-people-results {
            display: none;

            position: absolute;

            z-index: 20;

            top: calc(100% + 6px);
            left: 0;
            right: 0;

            max-height: 220px;

            overflow-y: auto;

            background: white;

            border:
                1px solid #dce5e2;

            border-radius: 10px;

            box-shadow:
                0 12px 30px rgba(20,
                    55,
                    47,
                    0.12);
        }


        .schedule-meeting-selected-people {
            display: flex;

            flex-wrap: wrap;

            gap: 8px;

            margin-top: 11px;
        }


        .schedule-meeting-no-people {
            color: #83918d;

            font-size: 12px;
        }


        .schedule-meeting-message {
            min-height: 18px;

            color: #a33f3f;

            font-size: 12px;
        }


        .schedule-meeting-footer {
            display: flex;

            align-items: center;
            justify-content: flex-end;

            gap: 10px;

            padding: 15px 22px;

            border-top:
                1px solid #e7ecea;

            background: #fbfcfc;
        }

        /* =================================================
   NOTIFICATION SEARCH
================================================= */

        .notification-search-wrapper {
            position: relative;

            width: 100%;
            max-width: 520px;

            margin-bottom: 18px;
        }


        .notification-search-icon {
            position: absolute;

            left: 15px;
            top: 50%;

            transform: translateY(-50%);

            width: 18px;
            height: 18px;

            color: #758782;

            pointer-events: none;
        }


        .notification-search {
            width: 100%;
            height: 44px;

            padding:
                0 42px 0 44px;

            border:
                1px solid #d6e0dd;

            border-radius: 11px;

            outline: none;

            background: white;
            color: #1c332e;

            font-size: 14px;

            transition:
                border-color 0.18s ease,
                box-shadow 0.18s ease;
        }


        .notification-search::placeholder {
            color: #91a09c;
        }


        .notification-search:hover {
            border-color: #bccdc8;
        }


        .notification-search:focus {
            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61, 145, 124, 0.11);
        }


        .notification-search-wrapper:focus-within .notification-search-icon {
            color: #25745f;
        }


        .notification-search::-webkit-search-cancel-button {
            display: none;
        }


        .notification-search-clear {
            position: absolute;

            right: 10px;
            top: 50%;

            transform: translateY(-50%);

            width: 27px;
            height: 27px;

            border: 0;
            border-radius: 7px;

            background: transparent;

            color: #71817d;

            display: none;

            align-items: center;
            justify-content: center;

            font-size: 18px;

            cursor: pointer;
        }


        .notification-search-clear:hover {
            background: #edf3f1;

            color: #244a41;
        }

        /* =================================================
   NOTIFICATION SEARCH HIGHLIGHT
================================================= */

        .notification-search-highlight {
            background: rgba(255, 193, 7, 0.28);
            color: inherit;

            padding: 0;
            margin: 0;

            border-radius: 2px;

            box-decoration-break: clone;
            -webkit-box-decoration-break: clone;
        }

        /* =================================================
   NOTIFICATIONS
================================================= */

        .notification-nav-button {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 10px;
        }


        .notification-unread-badge {
            min-width: 20px;
            height: 20px;

            padding: 0 6px;

            border-radius: 999px;

            background: #55d6a9;
            color: #10332c;

            align-items: center;
            justify-content: center;

            font-size: 10px;
            font-weight: 800;
        }


        .notification-toolbar {
            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 16px;

            margin-bottom: 18px;
        }


        .notification-filters {
            display: flex;

            align-items: center;
            flex-wrap: wrap;

            gap: 7px;
        }


        .notification-filter {
            border:
                1px solid #d6e0dd;

            background: white;
            color: #506660;

            padding: 8px 13px;

            border-radius: 999px;

            font-size: 12px;
            font-weight: 650;

            cursor: pointer;

            transition:
                background 0.15s ease,
                border-color 0.15s ease,
                color 0.15s ease;
        }


        .notification-filter:hover {
            border-color: #b8cbc5;

            background: #f5f9f7;

            color: #235d4f;
        }


        .notification-filter.active {
            border-color: #126e5a;

            background: #126e5a;
            color: white;
        }


        .notifications-list {
            display: grid;

            gap: 10px;
        }


        .notification-card {
            position: relative;

            display: flex;

            align-items: flex-start;

            gap: 14px;

            padding: 17px 18px;

            background: white;

            border:
                1px solid #e0e7e5;

            border-radius: 12px;

            transition:
                border-color 0.15s ease,
                box-shadow 0.15s ease,
                background 0.15s ease;
        }


        .notification-card:hover {
            border-color: #c8d7d3;

            box-shadow:
                0 3px 12px rgba(23,
                    51,
                    45,
                    0.045);
        }


        .notification-card.unread {
            background: #f5fbf8;

            border-color: #c9e4dc;
        }


        .notification-icon {
            width: 40px;
            height: 40px;

            flex-shrink: 0;

            border-radius: 11px;

            background: #edf4f2;
            color: #276656;

            display: flex;

            align-items: center;
            justify-content: center;

            font-size: 18px;
        }


        .notification-body {
            min-width: 0;

            flex: 1;
        }


        .notification-title-row {
            display: flex;

            align-items: flex-start;
            justify-content: space-between;

            gap: 15px;
        }


        .notification-title {
            color: #213d36;

            font-size: 14px;
            font-weight: 700;
        }


        .notification-time {
            flex-shrink: 0;

            color: #879590;

            font-size: 11px;
        }


        .notification-message {
            margin-top: 5px;

            color: #667873;

            font-size: 13px;

            line-height: 1.5;

            word-break: break-word;
        }


        .notification-actions {
            display: flex;

            align-items: center;
            flex-wrap: wrap;

            gap: 8px;

            margin-top: 11px;
        }


        .notification-unread-dot {
            width: 8px;
            height: 8px;

            flex-shrink: 0;

            margin-top: 7px;

            border-radius: 50%;

            background: #20a27c;
        }


        .notification-empty {
            padding: 45px 25px;

            background: white;

            border:
                1px dashed #ccd8d4;

            border-radius: 12px;

            text-align: center;

            color: #75837f;
        }


        /* Small action button used inside notification cards */

        .notification-action-button {
            border:
                1px solid #cfdad7;

            background: white;
            color: #25463f;

            padding: 7px 11px;

            border-radius: 7px;

            font-size: 11px;
            font-weight: 650;

            cursor: pointer;

            transition:
                background 0.15s ease,
                border-color 0.15s ease;
        }


        .notification-action-button:hover {
            background: #f1f7f4;

            border-color: #b8cbc5;
        }


        .notification-action-button.primary {
            border-color: #126e5a;

            background: #126e5a;
            color: white;
        }


        .notification-action-button.primary:hover {
            background: #0e5e4d;
        }


        /* Read notifications remain visible,
   but visually quieter. */

        .notification-card.read {
            background: white;

            border-color: #e4eae8;
        }


        .notification-card.read .notification-title {
            font-weight: 600;
        }


        .notification-card.read .notification-icon {
            background: #f1f4f3;

            color: #70817c;
        }

        /* =================================================
   NOTIFICATION ARRIVAL ANIMATIONS
================================================= */

        @keyframes notificationNavPulse {

            0% {
                box-shadow:
                    0 0 0 0 rgba(85, 214, 169, 0);
            }

            25% {
                background:
                    rgba(85, 214, 169, 0.18);

                box-shadow:
                    0 0 0 4px rgba(85, 214, 169, 0.08);
            }

            55% {
                background:
                    rgba(85, 214, 169, 0.08);
            }

            100% {
                box-shadow:
                    0 0 0 0 rgba(85, 214, 169, 0);
            }
        }


        .notification-nav-button.notification-arrived {
            animation:
                notificationNavPulse 0.7s ease 3;
        }


        @keyframes notificationCardArrival {

            from {
                opacity: 0;

                transform:
                    translateY(-7px);
            }

            to {
                opacity: 1;

                transform:
                    translateY(0);
            }
        }


        .notification-card-arriving {
            animation:
                notificationCardArrival 0.28s ease-out;
        }

        /* =================================================
   NOTIFICATION RESPONSIVE
================================================= */

        @media (max-width: 760px) {

            .notification-toolbar {
                align-items: stretch;

                flex-direction: column;
            }


            .notification-filters {
                width: 100%;
            }


            .notification-title-row {
                flex-direction: column;

                gap: 4px;
            }


            .notification-time {
                flex-shrink: 1;
            }


            .notification-card {
                padding: 15px;
            }

        }

        /* =================================================
   CHAT
================================================= */

        .chat-page-heading {
            margin-bottom: 18px;
        }


        .chat-workspace {
            height: calc(100vh - 165px);

            display: grid;
            grid-template-columns:
                minmax(260px, 320px) minmax(0, 1fr);

            overflow: hidden;

            background: white;

            border:
                1px solid #dfe7e4;

            border-radius: 14px;

            box-shadow:
                0 2px 8px rgba(23, 51, 45, 0.035);
        }


        /* =================================================
   CHAT SIDEBAR
================================================= */

        .chat-sidebar {
            min-width: 0;

            display: flex;
            flex-direction: column;

            background: #fbfcfc;

            border-right:
                1px solid #e3e9e7;
        }


        .chat-search-section {
            position: relative;

            padding: 16px;

            border-bottom:
                1px solid #e7ecea;
        }


        .chat-search-wrapper {
            position: relative;
        }


        .chat-search-icon {
            position: absolute;

            left: 13px;
            top: 50%;

            width: 17px;
            height: 17px;

            transform:
                translateY(-50%);

            color: #758782;

            pointer-events: none;
        }


        .chat-search-input {
            width: 100%;
            height: 42px;

            padding:
                0 13px 0 40px;

            border:
                1px solid #d6e0dd;

            border-radius: 10px;

            outline: none;

            background: white;
            color: #1c332e;

            font-size: 13px;

            transition:
                border-color 0.18s ease,
                box-shadow 0.18s ease;
        }


        .chat-search-input::placeholder {
            color: #91a09c;
        }


        .chat-search-input:focus {
            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61, 145, 124, 0.11);
        }


        .chat-search-wrapper:focus-within .chat-search-icon {
            color: #25745f;
        }


        .chat-search-input::-webkit-search-cancel-button {
            display: none;
        }


        /* =================================================
   PEOPLE SEARCH RESULTS
================================================= */

        .chat-people-search-results {
            position: absolute;

            z-index: 50;

            top: 66px;
            left: 16px;
            right: 16px;

            max-height: 300px;

            overflow-y: auto;

            background: white;

            border:
                1px solid #dce5e2;

            border-radius: 11px;

            box-shadow:
                0 14px 35px rgba(16, 47, 43, 0.15);
        }


        /* =================================================
   CONVERSATIONS
================================================= */

        .chat-conversations-header {
            padding:
                15px 17px 10px;

            color: #526963;

            font-size: 12px;
            font-weight: 750;

            text-transform: uppercase;

            letter-spacing: 0.04em;
        }


        .chat-conversation-list {
            flex: 1;

            min-height: 0;

            overflow-y: auto;

            padding:
                0 8px 12px;
        }


        .chat-conversation-empty {
            padding:
                38px 18px;

            text-align: center;

            color: #7c8c87;
        }


        .chat-conversation-empty-icon {
            margin-bottom: 10px;

            font-size: 25px;
        }


        .chat-conversation-empty-title {
            margin-bottom: 5px;

            color: #405b54;

            font-size: 13px;
            font-weight: 700;
        }


        .chat-conversation-empty-text {
            font-size: 12px;
            line-height: 1.55;
        }


        /* =================================================
   CHAT MAIN AREA
================================================= */

        .chat-main {
            min-width: 0;
            min-height: 0;

            display: flex;

            background: white;
        }


        .chat-empty-state {
            width: 100%;

            display: flex;
            flex-direction: column;

            align-items: center;
            justify-content: center;

            padding: 40px;

            text-align: center;

            color: #71827d;
        }


        .chat-empty-icon {
            width: 64px;
            height: 64px;

            margin-bottom: 18px;

            display: flex;
            align-items: center;
            justify-content: center;

            border-radius: 18px;

            background: #edf5f2;

            font-size: 27px;
        }


        .chat-empty-state h2 {
            margin:
                0 0 8px;

            color: #29463f;

            font-size: 20px;
        }


        .chat-empty-state p {
            max-width: 390px;

            margin: 0;

            color: #7a8b86;

            font-size: 13px;
            line-height: 1.6;
        }


        /* =================================================
   ACTIVE CONVERSATION
================================================= */

        .chat-conversation-panel {
            width: 100%;
            min-width: 0;
            min-height: 0;

            display: flex;
            flex-direction: column;
        }


        .chat-conversation-header {
            min-height: 72px;

            display: flex;

            align-items: center;
            justify-content: space-between;

            gap: 20px;

            padding:
                12px 18px;

            border-bottom:
                1px solid #e7ecea;
        }


        .chat-user-information {
            min-width: 0;

            display: flex;
            align-items: center;

            gap: 11px;
        }


        .chat-user-avatar {
            width: 40px;
            height: 40px;

            flex-shrink: 0;

            display: flex;
            align-items: center;
            justify-content: center;

            border-radius: 50%;

            background: #dff3ec;
            color: #17634f;

            font-size: 14px;
            font-weight: 800;
        }


        .chat-user-details {
            min-width: 0;
        }


        .chat-user-name {
            overflow: hidden;

            color: #213d36;

            font-size: 14px;
            font-weight: 700;

            text-overflow: ellipsis;
            white-space: nowrap;
        }


        .chat-user-presence {
            display: flex;

            align-items: center;

            gap: 6px;

            margin-top: 4px;

            color: #758782;

            font-size: 11px;
        }


        .chat-user-presence-dot {
            width: 8px;
            height: 8px;

            flex-shrink: 0;

            border-radius: 50%;
        }


        .chat-user-presence-dot.available {
            background: #22a06b;
        }


        .chat-user-presence-dot.busy,
        .chat-user-presence-dot.in-call {
            background: #d94c4c;
        }


        .chat-user-presence-dot.away {
            background: #e0a21a;
        }


        .chat-user-presence-dot.out-of-office {
            background: #7c63c7;
        }


        .chat-user-presence-dot.offline {
            background: #7d8b87;
        }


        /* =================================================
   HEADER ACTIONS
================================================= */

        .chat-header-actions {
            display: flex;

            align-items: center;

            gap: 7px;

            flex-shrink: 0;
        }


        .chat-header-action {
            width: 38px;
            height: 38px;

            display: flex;
            align-items: center;
            justify-content: center;

            border:
                1px solid #d7e1de;

            border-radius: 10px;

            background: white;
            color: #36564e;

            cursor: pointer;

            transition:
                background 0.15s ease,
                border-color 0.15s ease,
                transform 0.15s ease;
        }


        .chat-header-action:hover {
            background: #f1f7f4;

            border-color: #b9cec8;
        }


        .chat-header-action:active {
            transform:
                scale(0.96);
        }


        .chat-call-action {
            background: #126e5a;
            border-color: #126e5a;

            color: white;
        }


        .chat-call-action:hover {
            background: #0e5e4d;
            border-color: #0e5e4d;
        }


        /* =================================================
   MESSAGES
================================================= */

        .chat-messages {
            flex: 1;

            min-height: 0;

            overflow-y: auto;

            padding: 22px;

            background: #fafcfb;
        }


        .chat-message-placeholder {
            height: 100%;

            display: flex;

            align-items: center;
            justify-content: center;

            color: #93a09c;

            text-align: center;

            font-size: 12px;
        }


        /* =================================================
   MESSAGE COMPOSER
================================================= */

        .chat-composer {
            display: flex;

            align-items: flex-end;

            gap: 9px;

            padding: 13px 15px;

            border-top:
                1px solid #e7ecea;

            background: white;
        }


        .chat-composer-actions {
            display: flex;

            align-items: center;

            gap: 3px;
        }


        .chat-composer-action {
            width: 35px;
            height: 35px;

            border: 0;

            border-radius: 8px;

            background: transparent;

            cursor: pointer;

            font-size: 17px;
        }


        .chat-composer-action:hover:not(:disabled) {
            background: #edf3f1;
        }


        .chat-composer-action:disabled {
            opacity: 0.45;

            cursor: default;
        }


        .chat-message-input {
            flex: 1;

            min-width: 0;

            min-height: 40px;
            max-height: 120px;

            padding:
                10px 12px;

            border:
                1px solid #d6e0dd;

            border-radius: 10px;

            outline: none;

            resize: none;

            font-family: inherit;
            font-size: 13px;

            line-height: 1.45;

            color: #29463f;
        }


        .chat-message-input:focus {
            border-color: #3d917c;

            box-shadow:
                0 0 0 3px rgba(61, 145, 124, 0.09);
        }


        .chat-message-input:disabled {
            background: #f6f8f7;

            cursor: default;
        }


        .chat-send-button {
            min-width: 65px;
            height: 40px;

            padding:
                0 14px;

            border: 0;

            border-radius: 9px;

            background: #126e5a;
            color: white;

            font-size: 12px;
            font-weight: 700;

            cursor: pointer;
        }


        .chat-send-button:hover:not(:disabled) {
            background: #0e5e4d;
        }


        .chat-send-button:disabled {
            opacity: 0.45;

            cursor: default;
        }


        /* =================================================
   CHAT RESPONSIVE
================================================= */

        @media (max-width: 850px) {

            .chat-workspace {
                grid-template-columns:
                    240px minmax(0, 1fr);
            }

        }

        /* =================================================
           SETTINGS
        ================================================= */

        .settings-card {
            max-width: 650px;
        }


        .field-label {
            display: block;

            margin-bottom: 8px;

            font-size: 13px;
            font-weight: 650;
        }


        .text-input {
            width: 100%;

            padding: 12px 13px;

            border:
                1px solid #ccd8d4;

            border-radius: 8px;

            outline: none;
        }


        .text-input:focus {
            border-color: #3c8b78;
        }


        .settings-actions {
            display: flex;

            flex-wrap: wrap;

            gap: 9px;

            margin-top: 15px;
        }


        #instanceMessage {
            margin-top: 16px;

            color: #526762;

            font-size: 13px;
        }


        /* =================================================
           PLACEHOLDER PAGES
        ================================================= */

        .placeholder {
            padding: 40px;

            text-align: center;

            color: #71807d;
        }


        /* =================================================
           RESPONSIVE
        ================================================= */

        @media (max-width: 760px) {

            .sidebar {
                width: 175px;
                min-width: 175px;
            }


            .content {
                padding: 20px;
            }


            .meeting-card {
                align-items: flex-start;

                flex-direction: column;
            }


            .meeting-actions {
                width: 100%;

                flex-wrap: wrap;
            }


            .meetings-toolbar {
                align-items: stretch;

                flex-direction: column;
            }


            .meeting-search-wrapper {
                max-width: none;
            }

            .meeting-status-filter-wrapper {
                justify-content: flex-end;
            }


            .meeting-status-filter {
                flex: 1;
            }

            .meeting-details-modal {
                padding: 16px;
            }


            .meeting-details-dialog {
                max-height:
                    calc(100vh - 32px);
            }


            .meeting-details-grid {
                grid-template-columns: 1fr;
            }


            .meeting-details-participant {
                align-items: flex-start;

                flex-direction: column;
            }


            .meeting-details-participant-statuses {
                justify-content: flex-start;
            }

            .schedule-meeting-modal {
                padding: 16px;
            }


            .schedule-meeting-dialog {
                max-height:
                    calc(100vh - 32px);
            }


            .schedule-meeting-time-grid {
                grid-template-columns: 1fr;
            }
        }

        /* =====================================================
   NOTIFICATION LIST / DETAIL TRANSITION
===================================================== */

        #notificationsView {
            position: relative;
            overflow: hidden;
        }


        /*
 * Both notification screens use the
 * same content area.
 */
        .notification-panel {
            width: 100%;
        }


        /* -----------------------------------------------------
   LIST PANEL
----------------------------------------------------- */

        .notification-list-panel {
            transform: translateX(0);
            opacity: 1;

            transition:
                transform 220ms ease,
                opacity 180ms ease;
        }


        /*
 * Move the list slightly left when
 * notification details are open.
 */
        .notification-list-panel.detail-open {
            display: none;
            transform: translateX(-35px);
            opacity: 0;
            pointer-events: none;
        }


        /* -----------------------------------------------------
   DETAIL PANEL
----------------------------------------------------- */

        .notification-detail-panel {
            display: none;

            transform: translateX(45px);
            opacity: 0;
        }


        /*
 * Temporary visible state.
 *
 * JavaScript will add this when a
 * notification is opened.
 */
        .notification-detail-panel.active {
            display: block;

            animation:
                notificationDetailEnter 220ms ease forwards;
        }


        @keyframes notificationDetailEnter {

            from {
                transform: translateX(45px);
                opacity: 0;
            }

            to {
                transform: translateX(0);
                opacity: 1;
            }
        }


        /* -----------------------------------------------------
   BACK BUTTON
----------------------------------------------------- */

        .notification-detail-back {
            appearance: none;

            border: 0;
            background: transparent;

            padding: 0;
            margin: 0 0 26px 0;

            font-size: 14px;
            font-weight: 600;

            cursor: pointer;

            color: inherit;

            opacity: 0.72;

            transition:
                opacity 150ms ease,
                transform 150ms ease;
        }


        .notification-detail-back:hover {
            opacity: 1;
            transform: translateX(-2px);
        }


        /* -----------------------------------------------------
   DETAIL HEADER
----------------------------------------------------- */

        .notification-detail-header {
            display: flex;
            align-items: flex-start;

            gap: 18px;

            padding-bottom: 24px;

            border-bottom:
                1px solid rgba(0, 0, 0, 0.08);
        }


        .notification-detail-icon {
            width: 48px;
            height: 48px;

            flex: 0 0 48px;

            display: flex;
            align-items: center;
            justify-content: center;

            border-radius: 14px;

            font-size: 22px;

            background:
                rgba(0, 0, 0, 0.045);
        }


        .notification-detail-heading {
            min-width: 0;
        }


        .notification-detail-type {
            margin-bottom: 6px;

            font-size: 12px;
            font-weight: 700;

            text-transform: uppercase;

            letter-spacing: 0.06em;

            opacity: 0.55;
        }


        .notification-detail-title {
            margin: 0;

            font-size: 24px;
            line-height: 1.3;

            font-weight: 700;
        }


        .notification-detail-time {
            margin-top: 8px;

            font-size: 13px;

            opacity: 0.58;
        }


        /* -----------------------------------------------------
   MESSAGE
----------------------------------------------------- */

        .notification-detail-content {
            padding: 28px 0;
        }


        .notification-detail-message {
            margin: 0;

            max-width: 760px;

            font-size: 15px;
            line-height: 1.75;

            white-space: pre-wrap;
        }


        /* -----------------------------------------------------
   ACTIONS
----------------------------------------------------- */

        .notification-detail-actions {
            display: flex;

            align-items: center;

            gap: 10px;

            flex-wrap: wrap;

            margin-top: 8px;
        }

        /* =====================================================
   NOTIFICATION BACK TRANSITION
===================================================== */

        .notification-detail-panel.closing {
            display: block;

            animation:
                notificationDetailExit 180ms ease forwards;
        }


        @keyframes notificationDetailExit {

            from {
                transform: translateX(0);
                opacity: 1;
            }

            to {
                transform: translateX(35px);
                opacity: 0;
            }
        }


        /*
 * Smoothly restore notification list.
 */
        .notification-list-panel.returning {
            display: block;

            animation:
                notificationListReturn 220ms ease forwards;
        }


        @keyframes notificationListReturn {

            from {
                transform: translateX(-35px);
                opacity: 0;
            }

            to {
                transform: translateX(0);
                opacity: 1;
            }
        }
    </style>

</head>


<body>

    <div class="app">


        <!-- =============================================
             SIDEBAR
        ============================================== -->

        <aside class="sidebar">


            <div class="brand">

                <div class="brand-mark">
                    S
                </div>

                <div class="brand-name">
                    ServiceCall
                </div>

            </div>


            <nav class="navigation">

                <button class="nav-button active" data-view="homeView">
                    Home
                </button>

                <!-- <button class="nav-button" data-view="peopleView">
                    People
                </button> -->

                <button class="nav-button" data-view="meetingsView">
                    Meetings
                </button>

                <button class="nav-button notification-nav-button" data-view="notificationsView">

                    <span>
                        Notifications
                    </span>

                    <span id="notificationUnreadBadge" class="notification-unread-badge" style="display: none;">
                        0
                    </span>

                </button>

                <button class="nav-button" data-view="chatView">
                    Chat
                </button>

                <button class="nav-button" data-view="historyView">
                    History
                </button>

                <button class="nav-button" data-view="recordingsView">
                    Recordings
                </button>

            </nav>


            <div class="sidebar-bottom">


                <button class="nav-button" data-view="settingsView">
                    Settings
                </button>


            </div>


        </aside>


        <!-- =============================================
             MAIN
        ============================================== -->

        <main class="main">


            <header class="topbar">


                <div id="currentPageTitle" class="topbar-title">
                    Home
                </div>


                <div class="topbar-right">


                    <!-- =========================================
             PRESENCE
        ========================================== -->

                    <div class="presence-control">


                        <button id="presenceButton" class="presence-button" type="button">

                            <span id="presenceDot" class="presence-dot available"></span>


                            <span id="presenceText">
                                Available
                            </span>


                            <span class="presence-chevron">
                                ▾
                            </span>

                        </button>


                        <!-- Presence dropdown -->

                        <div id="presenceMenu" class="presence-menu" style="display: none;">


                            <button class="presence-option" type="button" data-presence="available">
                                <span class="presence-option-dot available"></span>
                                Available
                            </button>


                            <button class="presence-option" type="button" data-presence="busy">
                                <span class="presence-option-dot busy"></span>
                                Busy
                            </button>


                            <button class="presence-option" type="button" data-presence="away">
                                <span class="presence-option-dot away"></span>
                                Away
                            </button>


                            <button class="presence-option" type="button" data-presence="out of office">
                                <span class="presence-option-dot out-of-office"></span>
                                Out of Office
                            </button>

                            <button class="presence-option" type="button" data-presence="offline">
                                <span class="presence-option-dot offline"></span>
                                Offline
                            </button>

                            <!-- OOF reason -->

                            <div id="oofReasonPanel" class="oof-reason-panel" style="display: none;">

                                <label class="oof-reason-label" for="oofReasonInput">
                                    Out of Office reason
                                </label>


                                <textarea id="oofReasonInput" class="oof-reason-input" maxlength="250" rows="3"
                                    placeholder="For example: On leave until Monday"></textarea>


                                <div class="oof-reason-actions">

                                    <button id="cancelOofButton" class="oof-reason-cancel" type="button">
                                        Cancel
                                    </button>


                                    <button id="saveOofButton" class="oof-reason-save" type="button">
                                        Save
                                    </button>

                                </div>

                            </div>

                            <div class="presence-reset-divider"></div>

                            <button id="resetPresenceButton" class="presence-reset-button" type="button">
                                <span class="presence-reset-icon">
                                    ↻
                                </span>

                                <span>
                                    Reset status
                                </span>
                            </button>


                        </div>


                    </div>


                    <!-- =========================================
             CONNECTION
        ========================================== -->

                    <div id="connectionPill" class="connection-pill">
                        Checking connection...
                    </div>


                </div>


            </header>

            <section class="content">


                <!-- =====================================
                     HOME
                ====================================== -->

                <div id="homeView" class="view active">


                    <div class="welcome-card">


                        <h1>
                            Welcome to ServiceCall
                        </h1>


                        <p>
                            Calls, meetings and communication
                            connected to your ServiceNow work.
                        </p>


                        <div class="home-actions">


                            <button id="openActiveCallButton" class="secondary-button">
                                Open Active Call
                            </button>


                        </div>


                    </div>


                </div>


                <!-- =====================================
                     PEOPLE
                ====================================== -->

                <div id="peopleView" class="view">


                    <div class="page-heading">


                        <div>


                            <h1>
                                People
                            </h1>


                            <p>
                                Find and call ServiceCall users.
                            </p>


                        </div>


                    </div>


                    <div class="card">

                        <div>

                            <label for="peopleSearchInput">
                                Find a ServiceCall user
                            </label>

                            <input id="peopleSearchInput" type="text" placeholder="Search by name, username or email"
                                autocomplete="off">

                        </div>


                        <div id="peopleSearchMessage" style="margin-top: 10px;"></div>


                        <div id="peopleSearchResults" style="margin-top: 16px;"></div>

                    </div>


                </div>


                <!-- =====================================
                     MEETINGS
                ====================================== -->

                <div id="meetingsView" class="view">


                    <div class="page-heading">


                        <div>


                            <h1>
                                Meetings
                            </h1>


                            <p>
                                Schedule, join and manage
                                ServiceCall meetings.
                            </p>


                        </div>


                        <button id="scheduleMeetingButton" class="primary-button" type="button">
                            + Schedule meeting
                        </button>


                    </div>

                    <!-- Meeting Search + Status Filter -->

                    <div class="meetings-toolbar">


                        <div class="meeting-search-wrapper">


                            <svg class="meeting-search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                                stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">

                                <circle cx="11" cy="11" r="7"></circle>

                                <line x1="16.65" y1="16.65" x2="21" y2="21"></line>

                            </svg>


                            <input id="meetingSearchInput" class="meeting-search" type="search"
                                placeholder="Search meetings by title, number or organizer..." autocomplete="off"
                                spellcheck="false">


                            <button id="meetingSearchClear" class="meeting-search-clear" type="button"
                                aria-label="Clear meeting search" title="Clear search">
                                ×
                            </button>


                        </div>


                        <div class="meeting-status-filter-wrapper">


                            <label class="meeting-status-filter-label" for="meetingStatusFilter">
                                Status
                            </label>


                            <select id="meetingStatusFilter" class="meeting-status-filter"
                                aria-label="Filter meetings by status">

                                <option value="">
                                    All
                                </option>

                                <option value="scheduled">
                                    Scheduled
                                </option>

                                <option value="in progress">
                                    In Progress
                                </option>

                                <option value="ended">
                                    Ended
                                </option>

                                <option value="cancelled">
                                    Cancelled
                                </option>

                            </select>


                        </div>


                    </div>
                    <!-- Meeting List -->

                    <div id="meetingsContainer" class="meetings-list">


                        <div class="loading">
                            Open Meetings to load your meetings.
                        </div>


                    </div>


                    <!-- Pagination -->

                    <div id="meetingPagination" class="meeting-pagination"></div>


                </div>

                <!-- =====================================
     NOTIFICATIONS
====================================== -->

                <div id="notificationsView" class="view">

                    <div id="notificationListPanel" class="notification-panel notification-list-panel active">

                        <div class="page-heading">


                            <div>

                                <h1>
                                    Notifications
                                </h1>

                                <p>
                                    Updates from your ServiceCall activity.
                                </p>

                            </div>


                            <button id="markAllNotificationsReadButton" class="secondary-button" type="button"
                                style="display: none;">

                                Mark all as read

                            </button>


                        </div>



                        <!-- =====================================
     NOTIFICATION SEARCH
====================================== -->

                        <div class="notification-search-wrapper">

                            <svg class="notification-search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                                stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                                <circle cx="11" cy="11" r="7"></circle>

                                <line x1="16.65" y1="16.65" x2="21" y2="21"></line>

                            </svg>


                            <input id="notificationSearchInput" class="notification-search" type="search"
                                placeholder="Search notifications..." autocomplete="off" spellcheck="false">


                            <button id="notificationSearchClear" class="notification-search-clear" type="button"
                                aria-label="Clear notification search" title="Clear search">
                                ×
                            </button>

                        </div>
                        <!-- =====================================
         NOTIFICATION TOOLBAR
    ====================================== -->

                        <div class="notification-toolbar">


                            <div class="notification-filters">

                                <button class="notification-filter active" type="button" data-notification-filter="all">

                                    All

                                </button>


                                <button class="notification-filter" data-notification-filter="unread">
                                    Unread
                                    <span id="notificationUnreadFilterCount" class="notification-filter-count"
                                        style="display: none;">
                                        0
                                    </span>
                                </button>


                                <button class="notification-filter" type="button" data-notification-filter="read">

                                    Read

                                </button>

                            </div>


                            <button id="refreshNotificationsButton" class="secondary-button" type="button">

                                Refresh

                            </button>


                        </div>


                        <!-- =====================================
         NOTIFICATION LIST
    ====================================== -->

                        <div id="notificationsContainer" class="notifications-list">


                            <div class="loading">
                                Open Notifications to load your notifications.
                            </div>


                        </div>


                        <!-- =====================================
         NOTIFICATION PAGINATION
    ====================================== -->

                        <div id="notificationPagination" class="meeting-pagination">
                        </div>
                    </div>

                    <!-- =====================================
     NOTIFICATION DETAIL
====================================== -->

                    <div id="notificationDetailPanel" class="notification-panel notification-detail-panel"
                        aria-hidden="true">

                        <button id="notificationDetailBackButton" class="notification-detail-back" type="button">
                            ← Back to Notifications
                        </button>


                        <div class="notification-detail-header">

                            <div id="notificationDetailIcon" class="notification-detail-icon">
                                🔔
                            </div>


                            <div class="notification-detail-heading">

                                <div id="notificationDetailType" class="notification-detail-type">
                                    Notification
                                </div>


                                <h2 id="notificationDetailTitle" class="notification-detail-title">
                                    Notification
                                </h2>


                                <div id="notificationDetailTime" class="notification-detail-time">
                                </div>

                            </div>

                        </div>


                        <div class="notification-detail-content">

                            <p id="notificationDetailMessage" class="notification-detail-message">
                            </p>

                        </div>


                        <div id="notificationDetailActions" class="notification-detail-actions">
                        </div>

                    </div>

                </div>

                <!-- =====================================
     CHAT
====================================== -->

                <div id="chatView" class="view">

                    <div class="page-heading chat-page-heading">

                        <div>
                            <h1>
                                Chat
                            </h1>

                            <p>
                                Messages, calls and conversations with
                                ServiceCall users.
                            </p>
                        </div>

                    </div>


                    <div class="chat-workspace">

                        <!-- =============================
             LEFT SIDE
        ============================== -->

                        <aside class="chat-sidebar">

                            <div class="chat-search-section">

                                <div class="chat-search-wrapper">

                                    <svg class="chat-search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                                        stroke-width="2" stroke-linecap="round" stroke-linejoin="round"
                                        aria-hidden="true">
                                        <circle cx="11" cy="11" r="7"></circle>

                                        <line x1="16.65" y1="16.65" x2="21" y2="21"></line>
                                    </svg>


                                    <input id="chatPeopleSearchInput" class="chat-search-input" type="search"
                                        placeholder="Search people..." autocomplete="off" spellcheck="false">

                                </div>


                                <!-- Search results will be rendered here later -->

                                <div id="chatPeopleSearchResults" class="chat-people-search-results"
                                    style="display: none;"></div>

                            </div>

                            <!-- SAVED CHAT CONVERSATIONS -->

                            <div id="chatConversationList" class="chat-conversation-list"></div>


                            <div class="chat-conversations-header">

                                <span>
                                    Conversations
                                </span>

                            </div>


                            <!-- Real conversations will be rendered here -->

                            <div id="chatConversationList" class="chat-conversation-list">

                                <div class="chat-conversation-empty">

                                    <div class="chat-conversation-empty-icon">
                                        💬
                                    </div>

                                    <div class="chat-conversation-empty-title">
                                        No conversations yet
                                    </div>

                                    <div class="chat-conversation-empty-text">
                                        Search for someone above to start a conversation.
                                    </div>

                                </div>

                            </div>

                        </aside>


                        <!-- =============================
             RIGHT SIDE
        ============================== -->

                        <section class="chat-main">

                            <!-- Empty state shown until somebody is selected -->

                            <div id="chatEmptyState" class="chat-empty-state">

                                <div class="chat-empty-icon">
                                    💬
                                </div>

                                <h2>
                                    Select a conversation
                                </h2>

                                <p>
                                    Search for a ServiceCall user or select an
                                    existing conversation to start chatting.
                                </p>

                            </div>


                            <!--
                This becomes visible when a user/conversation
                is selected. We will wire it in renderer.js next.
            -->

                            <div id="chatConversationPanel" class="chat-conversation-panel" style="display: none;">

                                <!-- Conversation header -->

                                <header class="chat-conversation-header">

                                    <div class="chat-user-information">

                                        <div id="chatUserAvatar" class="chat-user-avatar">
                                            ?
                                        </div>


                                        <div class="chat-user-details">

                                            <div id="chatUserName" class="chat-user-name">
                                                User
                                            </div>


                                            <div class="chat-user-presence">

                                                <span id="chatUserPresenceDot"
                                                    class="chat-user-presence-dot offline"></span>

                                                <span id="chatUserPresenceText">
                                                    Offline
                                                </span>

                                            </div>

                                        </div>

                                    </div>


                                    <div class="chat-header-actions">

                                        <!-- Calendar / availability -->

                                        <button id="chatCalendarButton" class="chat-header-action" type="button"
                                            title="View availability" aria-label="View availability">
                                            <span aria-hidden="true">
                                                📅
                                            </span>
                                        </button>


                                        <!-- Direct call -->

                                        <button id="chatCallButton" class="chat-header-action chat-call-action"
                                            type="button" title="Call" aria-label="Call">
                                            <span aria-hidden="true">
                                                ☎
                                            </span>
                                        </button>

                                    </div>

                                </header>


                                <!-- Messages -->

                                <div id="chatMessages" class="chat-messages">

                                    <div class="chat-message-placeholder">

                                        <div>
                                            This is the beginning of your conversation.
                                        </div>

                                    </div>

                                </div>


                                <!-- Composer -->

                                <div class="chat-composer">

                                    <div class="chat-composer-actions">

                                        <button id="chatAttachButton" class="chat-composer-action" type="button"
                                            title="Attach file" disabled>
                                            📎
                                        </button>


                                        <button id="chatEmojiButton" class="chat-composer-action" type="button"
                                            title="Emoji" disabled>
                                            😊
                                        </button>

                                    </div>


                                    <textarea id="chatMessageInput" class="chat-message-input" rows="1"
                                        placeholder="Type a message..." disabled></textarea>


                                    <button id="chatSendButton" class="chat-send-button" type="button" disabled>
                                        Send
                                    </button>

                                </div>

                            </div>

                        </section>

                    </div>

                </div>

                <!-- =====================================
                     HISTORY
                ====================================== -->

                <div id="historyView" class="view">


                    <div class="page-heading">


                        <div>


                            <h1>
                                History
                            </h1>


                            <p>
                                Your previous ServiceCall activity.
                            </p>


                        </div>


                    </div>


                    <div class="card placeholder">
                        Call history will appear here.
                    </div>


                </div>


                <!-- =====================================
                     RECORDINGS
                ====================================== -->

                <div id="recordingsView" class="view">


                    <div class="page-heading">


                        <div>


                            <h1>
                                Recordings
                            </h1>


                            <p>
                                Your ServiceCall recording history.
                            </p>


                        </div>


                        <button class="secondary-button" id="refreshRecordingsButton" type="button">
                            Refresh
                        </button>


                    </div>


                    <div id="recordingsMessage" style="
                            margin-bottom:14px;
                            color:#40514d;
                        "></div>


                    <div id="recordingsList"></div>


                </div>



                <!-- =====================================
                     SETTINGS
                ====================================== -->

                <div id="settingsView" class="view">


                    <div class="page-heading">


                        <div>


                            <h1>
                                Settings
                            </h1>


                            <p>
                                Manage your ServiceNow connection.
                            </p>


                        </div>


                    </div>


                    <div class="card settings-card">


                        <form id="instanceForm">


                            <label class="field-label" for="instanceUrl">
                                ServiceNow instance
                            </label>


                            <input id="instanceUrl" class="text-input" type="text"
                                placeholder="https://dev12345.service-now.com">


                            <div class="settings-actions">


                                <button class="secondary-button" type="submit">
                                    Save Instance
                                </button>


                                <button class="primary-button" id="loginButton" type="button">
                                    Sign in to ServiceNow
                                </button>


                            </div>


                        </form>


                        <div id="instanceMessage"></div>

                        <!-- =========================================
     ACCOUNT SETTINGS
========================================== -->

                        <div id="accountSettingsSection" style="
        margin-top: 24px;
        padding-top: 20px;
        border-top: 1px solid #e1e7e5;
    ">

                            <div style="
            margin-bottom: 6px;
            font-size: 14px;
            font-weight: 700;
        ">
                                Account
                            </div>

                            <div style="
            margin-bottom: 16px;
            color: #687a76;
            font-size: 12px;
            line-height: 1.5;
        ">
                                Manage the ServiceNow account connected
                                to ServiceCall Desktop.
                            </div>


                            <!-- Current Account -->

                            <div id="currentAccountCard" style="
            padding: 14px;
            border: 1px solid #e1e7e5;
            border-radius: 10px;
            background: #fafcfb;
        ">

                                <div style="
                margin-bottom: 7px;
                color: #71827d;
                font-size: 11px;
                font-weight: 700;
                text-transform: uppercase;
            ">
                                    Current account
                                </div>


                                <div id="currentAccountName" style="
                color: #29463f;
                font-size: 14px;
                font-weight: 700;
            ">
                                    Connected ServiceNow user
                                </div>


                                <div id="currentAccountUsername" style="
                margin-top: 3px;
                color: #71827d;
                font-size: 12px;
            "></div>


                                <div id="currentAccountServiceCallId" style="
                margin-top: 3px;
                color: #71827d;
                font-size: 12px;
            "></div>


                                <div style="margin-top: 14px;">

                                    <button id="signOutButton" class="secondary-button" type="button">
                                        Sign Out
                                    </button>

                                </div>

                            </div>


                            <div id="accountMessage" style="
            margin-top: 10px;
            color: #526762;
            font-size: 12px;
        "></div>

                        </div>

                    </div>


                </div>


            </section>


        </main>


    </div>

    <!-- =============================================
     MEETING DETAILS MODAL
============================================== -->

    <div id="meetingDetailsModal" class="meeting-details-modal" aria-hidden="true">

        <div class="meeting-details-backdrop" data-meeting-details-close></div>


        <div class="meeting-details-dialog" role="dialog" aria-modal="true" aria-labelledby="meetingDetailsHeading">


            <!-- =====================================
             HEADER
        ====================================== -->

            <div class="meeting-details-header">

                <div>

                    <div class="meeting-details-heading">
                        Meeting Details
                    </div>

                    <div id="meetingDetailsNumber" class="meeting-details-number">
                        —
                    </div>

                </div>


                <button id="meetingDetailsCloseButton" class="meeting-details-close" type="button"
                    aria-label="Close meeting details" title="Close">
                    ×
                </button>

            </div>


            <!-- =====================================
             SCROLLABLE CONTENT
        ====================================== -->

            <div id="meetingDetailsContent" class="meeting-details-content">


                <!-- Title + Status -->

                <div class="meeting-details-title-row">

                    <h2 id="meetingDetailsHeading" class="meeting-details-title">
                        Meeting
                    </h2>


                    <span id="meetingDetailsStatus" class="meeting-status">
                        —
                    </span>

                </div>


                <!-- Description -->

                <div class="meeting-details-section">

                    <div class="meeting-details-label">
                        Description
                    </div>

                    <div id="meetingDetailsDescription" class="meeting-details-description">
                        No description.
                    </div>

                </div>


                <!-- =================================
                 MEETING INFORMATION
            ================================== -->

                <div class="meeting-details-section">

                    <div class="meeting-details-section-title">
                        Meeting information
                    </div>


                    <div class="meeting-details-grid">


                        <!-- Organizer -->

                        <div class="meeting-details-field">

                            <div class="meeting-details-label">
                                Organizer
                            </div>

                            <div id="meetingDetailsOrganizer" class="meeting-details-value">
                                —
                            </div>

                        </div>


                        <!-- Scheduled Start -->

                        <div class="meeting-details-field">

                            <div class="meeting-details-label">
                                Scheduled start
                            </div>

                            <div id="meetingDetailsStart" class="meeting-details-value">
                                —
                            </div>

                        </div>


                        <!-- Scheduled End -->

                        <div class="meeting-details-field">

                            <div class="meeting-details-label">
                                Scheduled end
                            </div>

                            <div id="meetingDetailsEnd" class="meeting-details-value">
                                —
                            </div>

                        </div>


                        <!-- Started By -->

                        <div id="meetingDetailsStartedByField" class="meeting-details-field">

                            <div class="meeting-details-label">
                                Started by
                            </div>

                            <div id="meetingDetailsStartedBy" class="meeting-details-value">
                                —
                            </div>

                        </div>


                        <!-- Started At -->

                        <div id="meetingDetailsStartedAtField" class="meeting-details-field">

                            <div class="meeting-details-label">
                                Started at
                            </div>

                            <div id="meetingDetailsStartedAt" class="meeting-details-value">
                                —
                            </div>

                        </div>


                        <!-- Ended At -->

                        <div id="meetingDetailsEndedAtField" class="meeting-details-field">

                            <div class="meeting-details-label">
                                Ended at
                            </div>

                            <div id="meetingDetailsEndedAt" class="meeting-details-value">
                                —
                            </div>

                        </div>


                    </div>

                </div>


                <!-- =================================
                 PARTICIPANTS
            ================================== -->

                <div class="meeting-details-section">

                    <div class="meeting-details-participants-header">

                        <div class="meeting-details-section-title">
                            Participants
                        </div>

                        <div id="meetingDetailsParticipantCount" class="meeting-details-count">
                            0
                        </div>

                    </div>


                    <div id="meetingDetailsParticipants" class="meeting-details-participants">

                        <div class="meeting-details-empty">
                            No participants.
                        </div>

                    </div>

                </div>


            </div>


            <!-- =====================================
             MEETING DETAILS FOOTER
        ====================================== -->

            <div class="meeting-details-footer">


                <!--
                Context-sensitive action.

                Scheduled:
                    Start Meeting

                In Progress:
                    Join Meeting

                Ended / Cancelled:
                    Hidden for now
            -->

                <button id="meetingDetailsActionButton" class="primary-button" type="button" style="display: none;">
                    Join Meeting
                </button>


                <button id="meetingDetailsFooterCloseButton" class="secondary-button" type="button">
                    Close
                </button>


            </div>


        </div>

    </div>



    <!-- =============================================
     SCHEDULE / EDIT MEETING MODAL
============================================== -->

    <div id="scheduleMeetingModal" class="schedule-meeting-modal" aria-hidden="true">

        <div class="schedule-meeting-backdrop" data-schedule-meeting-close></div>


        <div class="schedule-meeting-dialog" role="dialog" aria-modal="true" aria-labelledby="scheduleMeetingHeading">


            <!-- =====================================
             HEADER
        ====================================== -->

            <div class="schedule-meeting-header">

                <div>

                    <div id="scheduleMeetingHeading" class="schedule-meeting-heading">
                        Schedule Meeting
                    </div>


                    <div id="scheduleMeetingSubtitle" class="schedule-meeting-subtitle">
                        Create a new ServiceCall meeting.
                    </div>

                </div>


                <button id="scheduleMeetingCloseButton" class="schedule-meeting-close" type="button"
                    aria-label="Close schedule meeting" title="Close">
                    ×
                </button>

            </div>


            <!-- =====================================
             FORM CONTENT
        ====================================== -->

            <div class="schedule-meeting-content">


                <!-- Meeting Title -->

                <div class="schedule-meeting-field">

                    <label class="schedule-meeting-label" for="scheduleMeetingTitle">
                        Meeting title

                        <span class="schedule-meeting-required">
                            *
                        </span>
                    </label>


                    <input id="scheduleMeetingTitle" class="schedule-meeting-input" type="text" maxlength="160"
                        placeholder="Enter meeting title" autocomplete="off">

                </div>


                <!-- Description -->

                <div class="schedule-meeting-field">

                    <label class="schedule-meeting-label" for="scheduleMeetingDescription">
                        Description
                    </label>


                    <textarea id="scheduleMeetingDescription" class="schedule-meeting-textarea" rows="4"
                        placeholder="What is this meeting about?"></textarea>

                </div>


                <!-- =================================
                 DATE / TIME
            ================================== -->

                <div class="schedule-meeting-time-grid">


                    <!-- Start -->

                    <div class="schedule-meeting-field">

                        <label class="schedule-meeting-label" for="scheduleMeetingStart">
                            Start date &amp; time

                            <span class="schedule-meeting-required">
                                *
                            </span>
                        </label>


                        <input id="scheduleMeetingStart" class="schedule-meeting-input" type="datetime-local">

                    </div>


                    <!-- End -->

                    <div class="schedule-meeting-field">

                        <label class="schedule-meeting-label" for="scheduleMeetingEnd">
                            End date &amp; time

                            <span class="schedule-meeting-required">
                                *
                            </span>
                        </label>


                        <input id="scheduleMeetingEnd" class="schedule-meeting-input" type="datetime-local">

                    </div>


                </div>


                <!-- =================================
                 TIME ZONE
            ================================== -->

                <div class="schedule-meeting-timezone">

                    <span class="schedule-meeting-timezone-label">
                        Time zone:
                    </span>

                    <span id="scheduleMeetingTimezone">
                        Loading...
                    </span>

                </div>


                <!-- =================================
                 PEOPLE
            ================================== -->

                <div class="schedule-meeting-field">

                    <label class="schedule-meeting-label" for="scheduleMeetingPeopleSearch">
                        People

                        <span class="schedule-meeting-required">
                            *
                        </span>
                    </label>


                    <div class="schedule-meeting-people-search-wrapper">

                        <input id="scheduleMeetingPeopleSearch" class="schedule-meeting-input" type="search"
                            placeholder="Search people..." autocomplete="off" spellcheck="false">


                        <div id="scheduleMeetingPeopleResults" class="schedule-meeting-people-results"></div>

                    </div>


                    <div id="scheduleMeetingSelectedPeople" class="schedule-meeting-selected-people">

                        <div id="scheduleMeetingNoPeople" class="schedule-meeting-no-people">
                            No people selected.
                        </div>

                    </div>

                </div>


                <!-- =================================
                 MESSAGE
            ================================== -->

                <div id="scheduleMeetingMessage" class="schedule-meeting-message"></div>


            </div>


            <!-- =====================================
             SCHEDULE / EDIT FOOTER
        ====================================== -->

            <div class="schedule-meeting-footer">


                <button id="scheduleMeetingCancelButton" class="secondary-button" type="button">
                    Close
                </button>


                <button id="scheduleMeetingSubmitButton" class="primary-button" type="button">
                    Schedule Meeting
                </button>


            </div>


        </div>

    </div>

    <script src="renderer.js"></script>


</body>

</html>
