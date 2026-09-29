   /* =====================================================
   MESSAGE SELECTION INDICATOR
===================================================== */

    const selectionIndicator = document.createElement("button");

    selectionIndicator.type = "button";

    selectionIndicator.className = "chat-message-selection-indicator";

    selectionIndicator.title = "Select message";

    selectionIndicator.style.cssText = `
  position:absolute;
  top:50%;

  width:20px;
  height:20px;

  padding:0;

  display:flex;
  align-items:center;
  justify-content:center;

  transform:translateY(-50%);

  border:1.5px solid #aab8b4;
  border-radius:50%;

  background:#ffffff;
  color:transparent;

  font-size:12px;
  font-weight:700;

  cursor:pointer;

  opacity:0;
  pointer-events:none;

  transition:
    opacity 0.15s ease,
    background 0.15s ease,
    border-color 0.15s ease,
    transform 0.15s ease;
`;

    /*
     * Keep the selection control on the
     * opposite side of the message bubble.
     *
     * My message       -> checkbox on left
     * Received message -> checkbox on right
     */
    if (message.is_mine) {
      selectionIndicator.style.left = "8px";
      selectionIndicator.style.right = "auto";
    } else {
      selectionIndicator.style.right = "8px";
      selectionIndicator.style.left = "auto";
    }

    selectionIndicator.addEventListener("click", (event) => {
      event.preventDefault();
      event.stopPropagation();

      if (!chatMessageSelectionMode) {
        return;
      }

      toggleChatMessageSelection(messageRow, message);
    });

    messageRow.appendChild(selectionIndicator);

    const bubbleWrapper = document.createElement("div");

    bubbleWrapper.style.cssText = `
    display:flex;
    align-items:center;
    gap:6px;
    max-width:78%;
    position:relative;
  `;

    /*
     * Incoming:
     * [actions] [message]
     *
     * Outgoing:
     * [message] [actions]
     */
    bubbleWrapper.style.flexDirection = message.is_mine ? "row-reverse" : "row";

    const bubble = document.createElement("div");

    /* =====================================================
     REPLIED MESSAGE PREVIEW
  ===================================================== */

    const replyTo = message.reply_to || null;

    if (replyTo) {
      const replyPreview = document.createElement("div");

      replyPreview.className = "chat-message-reply-preview";

      replyPreview.title = "Go to original message";

      replyPreview.style.cssText = `
      margin-bottom:6px;
      padding:6px 8px;

      border-left:3px solid
        ${message.is_mine ? "#4f8f7d" : "#7b8d88"};

      border-radius:6px;

      background:
        ${
          message.is_mine ? "rgba(255,255,255,0.48)" : "rgba(255,255,255,0.72)"
        };

      overflow:hidden;
      cursor:pointer;
    `;

      const replySender = document.createElement("div");

      replySender.textContent = String(replyTo.sender_name || "Unknown User");

      replySender.style.cssText = `
      margin-bottom:2px;
      color:#397261;
      font-size:10.5px;
      font-weight:700;
      white-space:nowrap;
      overflow:hidden;
      text-overflow:ellipsis;
    `;

      const replyText = document.createElement("div");

      if (replyTo.deleted === true) {
        replyText.textContent = "Message deleted";

        replyText.style.fontStyle = "italic";
      } else {
        const originalText = String(replyTo.text || "");

        replyText.textContent =
          originalText.length > 140
            ? originalText.substring(0, 137) + "..."
            : originalText;
      }

      replyText.style.cssText += `
      color:#66736f;
      font-size:10.5px;
      line-height:1.35;
      white-space:nowrap;
      overflow:hidden;
      text-overflow:ellipsis;
    `;

      replyPreview.appendChild(replySender);

      replyPreview.appendChild(replyText);

      replyPreview.addEventListener("click", async (event) => {
        event.stopPropagation();

        const originalMessageSysId = String(replyTo.sys_id || "").trim();

        if (!originalMessageSysId) {
          return;
        }

        await jumpToChatMessage(originalMessageSysId);
      });

      bubble.appendChild(replyPreview);
    }

    /* =====================================================
     ACTUAL MESSAGE TEXT
  ===================================================== */

    const messageText = document.createElement("div");

    /*
     * Deleted-for-everyone messages are
     * represented by the backend as:
     *
     * deleted: true
     * text: ""
     */
    if (message.deleted === true) {
      messageText.textContent = "Message deleted";

      messageText.style.cssText = `
      white-space:pre-wrap;
      overflow-wrap:anywhere;
      font-style:italic;
      color:#7f8986;
    `;
    } else {
      messageText.textContent = message.text || "";

      messageText.style.cssText = `
      white-space:pre-wrap;
      overflow-wrap:anywhere;
    `;
    }

    bubble.appendChild(messageText);

    bubble.style.cssText = `
    max-width:100%;
    padding:9px 12px;
    border-radius:12px;
    font-size:13px;
    line-height:1.4;
    white-space:pre-wrap;
    overflow-wrap:anywhere;
    background:${message.is_mine ? "#dff3ec" : "#f1f3f2"};
    color:#1f2927;
  `;

    /* =====================================================
     REPLY BUTTON
  ===================================================== */

    const replyButton = document.createElement("button");

    replyButton.type = "button";
    replyButton.textContent = "↩";
    replyButton.title = "Reply";

    replyButton.style.cssText = `
    width:24px;
    height:24px;
    min-width:24px;
    border:none;
    border-radius:50%;
    background:#eef2f1;
    color:#60706c;
    font-size:14px;
    line-height:24px;
    padding:0;
    cursor:pointer;
    opacity:0;
    pointer-events:none;
    transition:
      opacity 0.15s ease,
      background 0.15s ease,
      transform 0.15s ease;
  `;

    /*
     * A persisted, non-deleted message is
     * required before it can become a
     * reply target.
     */
    if (!messageSysId || message.deleted === true) {
      replyButton.style.display = "none";
    }

    replyButton.addEventListener("mouseenter", () => {
      replyButton.style.background = "#dfe8e5";

      replyButton.style.transform = "scale(1.08)";
    });

    replyButton.addEventListener("mouseleave", () => {
      replyButton.style.background = "#eef2f1";

      replyButton.style.transform = "scale(1)";
    });

    replyButton.addEventListener("click", (event) => {
      event.stopPropagation();

      if (isActiveChatReadOnly()) {
        showChatMembershipError();
        return;
      }

      if (!messageSysId || message.deleted === true) {
        return;
      }

      chatReplyTarget = {
        sys_id: messageSysId,

        sender_name: String(
          message.sender_name ||
            message.sender_user_name ||
            (message.is_mine ? "You" : "Unknown User"),
        ),

        text: String(message.text || ""),
      };

      renderChatReplyPreview();

      if (chatMessageInput) {
        chatMessageInput.focus();
      }
    });
