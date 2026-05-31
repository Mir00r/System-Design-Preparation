## ✨ STAR Answer

---

### ⭐ **SITUATION**
After we deployed a new **transaction OTP feature** (like I just showed you), an end-user reported:
> "I'm not receiving the OTP sometimes, even though the system says it's sent."

This was happening intermittently — not every time.

---

### 📋 **TASK**
My job was to:
- **Investigate the root cause** quickly.
- **Communicate clearly** with users and stakeholders.
- **Fix the issue** permanently without introducing new bugs.
- **Prevent recurrence** with better monitoring.

---

### ⚡ **ACTION**

Here’s exactly how I handled it step-by-step:

✅ **Acknowledge First, Politely:**
- I immediately thanked the user for reporting it.
- Reassured them:
  > "Thank you for bringing this up! We are investigating it on priority. Your experience matters to us."

✅ **Gather Key Information Quickly:**
- Asked specific questions:
  > When did it happen?  
  > Any error message received?  
  > Was it a specific device or network?  
  > Any pattern (e.g., peak hours)?
- Pulled backend logs for that user's session ID and timestamps.

✅ **Replicate or Simulate:**
- Tried simulating the same scenario internally under similar conditions (high server load, slow network).

✅ **Root Cause Analysis:**
- Found that sometimes the **background cleaner thread** was expiring OTPs **too aggressively** (if system clock drifted slightly).
- Also discovered that **OTP delivery service (SMS API)** had a small timeout issue under heavy load.

✅ **Proactive Fix:**
- Added **extra buffer time (10 seconds grace)** in the OTP expiry.
- Added **retry mechanism** for SMS sending.
- Improved **observability**: added more detailed logging for OTP status per user.

✅ **Communicate Status Regularly:**
- Kept the user/stakeholders updated every few hours:
  > "We have identified a possible cause and are testing the fix internally. Thank you for your patience."

✅ **Deploy Patch Safely:**
- Hotfixed the expiry time logic via a safe, canary deployment.
- Monitored OTP success rates and error rates after fix.

✅ **Postmortem + Prevention:**
- Documented the issue and learning in an internal **RCAs (Root Cause Analysis) document**.
- Added **health-check alarms** for SMS delivery latency.
- Suggested we **simulate random network slowdowns** in future staging tests.

---

### 🏆 **RESULT**

- The user received the corrected OTPs perfectly after the fix.
- They actually sent a **thank you note** later! 🙏🏼
- System OTP delivery rate improved from **97% → 99.8%**.
- No further complaints on OTP expiry for months.

---

## 🎯 In short, **my natural reaction when a user complains**:

| Step | Action |
|:---|:---|
| 1 | **Acknowledge quickly** with empathy and politeness. |
| 2 | **Investigate logically** (data-driven, not assumption-driven). |
| 3 | **Communicate status** clearly and proactively. |
| 4 | **Implement safe fix** after validating root cause. |
| 5 | **Learn and improve** monitoring or coding for future resilience. |

---

## 🧠 Bonus Tip
✅ Always assume **positive intent** when users complain.  
✅ It's a chance to **make the system stronger**, **build user trust**, and **grow yourself**.  
✅ Never react emotionally; respond **professionally and logically**.

---

Would you also like me to show how I sometimes **prepare "emergency rollback plans"** before deploying urgent fixes? (kind of like a **Plan B** checklist 🚑✨)  
Let me know! 🔥  
(Just say "show rollback strategy") 🚀

---

*Previous: [← Logical Thinking Behind Coding](./LogicalThinkingBehindTheCoding.md) | Next: [Behavioral Q&A Deep Dive →](./Engineering/README.md)*
