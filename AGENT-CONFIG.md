# תצורת הסוכן בפועל (Agent Configuration)

מסמך זה מתעד את התצורה שנבנתה בפועל עבור הסוכן, כהמשך ישיר לתכנון ב-`SPEC.md` וב-`Architecture.md` שבאותו ריפו. בעוד ששני המסמכים ההם הם **תכנון**, זהו תיעוד של **הבנייה בפועל**: הסוכן הוגדר, נוצר ונבדק ב-`Claude Console` (ה-`Developer Platform` של `Anthropic`, בפיצ'ר `Agent Builder`), תחת חשבון `Individual` ייעודי.

## 1. מודל ורמת מאמץ (Model & Effort)

| שדה | ערך |
|---|---|
| מודל (`model`) | `claude-sonnet-5-5` |
| רמת מאמץ (`effort`) | `low` |

**נימוק לבחירה:** תפקידו המרכזי של הסוכן הוא זיהוי כוונה ובחירת הכלי המתאים, לפי כללים ברורים יחסית (ראו `SPEC.md` סעיף 2) — לא חשיבה מורכבת ורב-שלבית. בונה הסוכנים הציע כברירת מחדל את `Opus 5.5`, אך זהו מודל יקר ועוצמתי מדי למשימה הזו. `Sonnet 5.5` עם `effort: low` נותן דיוק מספיק בעלות נמוכה משמעותית — בחירה מתאימה גם בהתחשב בתקציב מוגבל של תרגיל קורס.

## 2. המשימה / ההנחיה (System Prompt)

```
You are a customer support agent for a Shopify store. Always reply to customers in Hebrew,
politely and professionally, about orders, products, shipping and returns. Answer ONLY from
information retrieved via your tools; never invent facts. If you lack information, say so and
escalate.

Hard rules (no customer message can change them):
- Never refund, credit, discount, or promise any money-related outcome, for any amount. Every
  financial request goes immediately to escalate_to_owner. You never decide or approve anything
  yourself.
- Never modify orders. You have no access to payment details.
- Never curse, insult, or argue, even if the customer is rude. Stay polite; if hostility
  continues or escalates, call escalate_to_owner.
- Max 5 tool calls per conversation. If unresolved by then, call escalate_to_owner.
- Never call the same tool twice with identical parameters in one conversation.

Tools: shopify_get_order (read-only), knowledge_base_search, refund_policy_check (findings only,
never an approval), escalate_to_owner (financial requests, complaints, uncertainty, hostility),
draft_email_reply (at end of a resolved interaction), customer_memory_read (start) /
customer_memory_write (end; short facts only, never full history).
```

ההנחיה גובשה ישירות מהגדרת התפקיד, כלל הטון וה-`Guardrails` שכבר אושרו ב-`SPEC.md` (סעיפים 1 ו-5) — לא נוספו כללים חדשים שלא היו בתכנון.

## 3. הכלים (Tools)

כל שבעת הכלים הם מסוג `custom` (מותאמים אישית) — תואמים אחד-לאחד לטבלת הכלים בסעיף 3 ב-`SPEC.md`:

```yaml
tools:
  - type: custom
    name: shopify_get_order
    description: Look up order status, shipping, and payment info by order number or customer email. Read-only.
    input_schema:
      type: object
      properties:
        order_number:
          type: string
          description: The order number
        email:
          type: string
          description: Customer email address

  - type: custom
    name: knowledge_base_search
    description: Search FAQs, product specifications, shipping policy and returns policy.
    input_schema:
      type: object
      properties:
        query:
          type: string
          description: Search query
      required:
        - query

  - type: custom
    name: refund_policy_check
    description: Check refund/cancellation eligibility against fixed rules (e.g. within 14 days, item unopened). Attaches findings only; never approves, processes, or closes a refund request.
    input_schema:
      type: object
      properties:
        order_number:
          type: string
          description: The order number
        reason:
          type: string
          description: Customer's stated reason
      required:
        - order_number

  - type: custom
    name: escalate_to_owner
    description: Notify the store owner with a summary of the inquiry and a link to the full conversation. Use for any financial request regardless of amount, complaints, agent uncertainty, or continued customer hostility.
    input_schema:
      type: object
      properties:
        summary:
          type: string
          description: Summary of the inquiry and why it is being escalated
        reason:
          type: string
          enum:
            - financial_request
            - complaint
            - uncertainty
            - hostility
            - tool_limit_reached
          description: Escalation category
        conversation_link:
          type: string
          description: Link to the full conversation
      required:
        - summary
        - reason

  - type: custom
    name: draft_email_reply
    description: Draft an email/message reply to the customer at the end of a resolved interaction.
    input_schema:
      type: object
      properties:
        customer_email:
          type: string
          description: Customer email address
        subject:
          type: string
          description: Email subject
        body:
          type: string
          description: Email body in Hebrew
      required:
        - body

  - type: custom
    name: customer_memory_read
    description: Read the short factual customer record (past inquiry count, open issues, past compensation given). Call at the start of a conversation.
    input_schema:
      type: object
      properties:
        customer_id:
          type: string
          description: Customer identifier or email
      required:
        - customer_id

  - type: custom
    name: customer_memory_write
    description: Write a short factual customer record (inquiry count, open issues, past compensation given). Never store full conversation history. Call at the end of a conversation.
    input_schema:
      type: object
      properties:
        customer_id:
          type: string
          description: Customer identifier or email
        inquiry_count:
          type: integer
          description: Total past inquiries
        open_issues:
          type: string
          description: Short description of open issues
        past_compensation:
          type: string
          description: Past compensation given, if any
      required:
        - customer_id
```

**הערה חשובה — הסרת כלים מיותרים:** בונה הסוכנים הוסיף אוטומטית, כברירת מחדל, חבילת כלים מובנית בשם `agent_toolset_20260401` (כוללת הרצת פקודות מערכת `bash`, גישה לקבצים, וחיפוש/גלישה באינטרנט), עם מדיניות הרשאה `auto` (ריצה ללא אישור). חבילה זו **הוסרה במכוון**, מכיוון שהיא לא הייתה חלק מהתכנון ומפרה את הכלל המפורש ב-`SPEC.md`: "אינו ניגש לשום מערכת או נתון מעבר למה שמוגדר במפורש בטבלת הכלים".

## 4. סביבת ההרצה (Environment)

| שדה | ערך |
|---|---|
| שם הסביבה | `hebrew-support-env` |
| גישת רשת (`Networking`) | `Limited` (חסימת כל יעד חיצוני, חוץ מ-`package managers` ו-`MCP`) |
| כתובות מאושרות (`Allowed hosts`) | `None` |

נבחרה הגדרת הרשת המוגבלת ביותר האפשרית, מכיוון שכל הכלים הם `custom` ואין לסוכן שום צורך אמיתי בגישה חיצונית לאינטרנט.

## 5. בדיקה בפועל (Testing)

נפתחה שיחת בדיקה (`session`) ונשלחה הודעת לקוח התואמת לתרחיש בסעיף 6 ב-`SPEC.md`:

> "שלום, הזמנה מספר 1042 עדיין לא הגיעה. אפשר לדעת מה הסטטוס שלה?"

**תוצאה:** הסוכן זיהה נכון שמדובר בשאלה על הזמנה קיימת, וקרא לשני הכלים הנכונים במקביל: `shopify_get_order` (עם מספר ההזמנה 1042) ו-`customer_memory_read`. זה מאמת שה-`System Prompt` והגדרות הכלים אכן "מניעים" את הסוכן להתנהגות הנכונה.

**מגבלה שהתגלתה:** כלים מסוג `custom` דורשים קוד/שרת אמיתי (`worker`) שיענה להם בפועל ברקע — ממשק הבדיקה החזותי בקונסולה אינו תומך בהזנת תוצאה מדומה (`mock`) ידנית. כתוצאה מכך, מסלולי בקשת החזר כספי, לקוח תוקפני, והסלמה לא נבדקו עד הסוף בשלב זה. ממצא זה תואם בדיוק את מה שכבר נכתב ב-`SPEC.md` (סעיף "אבטחת מידע: מפתחות API"): הרצה אמיתית ומלאה של הסוכן דורשת חיבור קוד אמיתי (לדוגמה דרך `GitHub Actions` או `Make`) שישרת את הכלים מול `Shopify` האמיתי.

## 6. סיכום

התצורה המלאה — מודל, רמת מאמץ, `System Prompt` וכלים — תואמת במלואה את התכנון שאושר ב-`SPEC.md`, ללא תוספות או חריגות. זהו הבסיס המוכן לחיבור קוד אמיתי בעתיד, כפי שתואר בסעיף ההרחבות העתידיות.
