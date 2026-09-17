<div class="topbar">
    ServiceCall Desktop
</div>

<div class="page">

    <div class="card">

        <h1>
            Connect ServiceNow
        </h1>

        <p>
            Enter the ServiceNow instance you want
            ServiceCall Desktop to connect to.
        </p>

        <form id="instanceForm">

            <input id="instanceUrl" type="text" placeholder="https://dev12345.service-now.com" style="
                    width:100%;
                    padding:14px;
                    border:1px solid #ccd8d4;
                    border-radius:8px;
                    font-size:14px;
                " />

            <button class="button" type="submit">
                Save Instance
            </button>

            <button class="button" id="loginButton" type="button">
                Sign in to ServiceNow
            </button>

            <button id="openActiveCallButton">
                Open Active Call
            </button>

        </form>

        <div id="instanceMessage" style="
                margin-top:18px;
                color:#40514d;
            ">
        </div>

    </div>

</div>

<script src="renderer.js"></script>
