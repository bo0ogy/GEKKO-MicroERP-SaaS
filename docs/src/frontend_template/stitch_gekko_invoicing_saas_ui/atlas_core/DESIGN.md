```markdown
# Design System Strategy: The Luminous Ledger

## 1. Overview & Creative North Star
This design system is built to transform the mundane nature of ERP and invoicing into a high-performance, editorial experience. We are moving away from the "cluttered dashboard" trope and toward a philosophy of **Luminous Precision**.

The **Creative North Star** for this system is "The Architectural Ghost." Like a high-end physical office in Casablanca—minimalist, dark, and filled with glass and light—the interface should feel structured yet weightless. We break the "template" look through intentional asymmetry: sidebars that don't reach the top, overlapping data visualizations, and high-contrast typography scales that prioritize information hierarchy over rigid grid boxes.

## 2. Colors & Surface Philosophy
The palette is rooted in deep obsidian tones, punctuated by a signature primary gradient that transitions from deep indigo to electric violet, reflecting the technological ambition of Moroccan micro-enterprises.

*   **The "No-Line" Rule:** To achieve a premium feel, designers are prohibited from using 1px solid borders for sectioning. Boundaries must be defined solely through background color shifts. For example, a `surface-container-low` (#131314) section should sit on a `background` (#0e0e0f) to create a soft, natural break.
*   **Surface Hierarchy & Nesting:** Treat the UI as a series of physical layers. 
    *   Base: `surface` (#0e0e0f)
    *   Sectioning: `surface-container-low` (#131314)
    *   Actionable Cards: `surface-container-highest` (#262627)
*   **The "Glass & Gradient" Rule:** Floating elements, such as navigation bars or active modals, must use glassmorphism. Combine `surface-variant` (#262627) at 60% opacity with a `backdrop-blur` of 20px. 
*   **Signature Textures:** Use the primary gradient (`primary` #a3a6ff to `secondary` #c180ff) sparingly. Reserve it for high-value CTAs or subtle "glow" backgrounds behind hero metrics to provide a sense of visual "soul."

## 3. Typography
We use a dual-font approach to balance authority with utility.

*   **Display & Headlines (Manrope):** This is our "Editorial" voice. Use `display-lg` (3.5rem) and `headline-md` (1.75rem) with tighter letter-spacing (-0.02em) to create an authoritative, modern presence for brand moments and key financial summaries.
*   **Body & UI (Inter):** For the "Utility" voice. Inter provides maximum legibility for complex invoicing tables. Use `body-md` (0.875rem) for standard text and `label-sm` (0.6875rem) in all-caps for metadata to give it a technical, "fintech" precision.

## 4. Elevation & Depth
Depth is a functional tool, not a decoration. We convey hierarchy through **Tonal Layering** rather than traditional drop shadows.

*   **The Layering Principle:** Stack `surface-container` tiers. Place a `surface-container-lowest` (#000000) card inside a `surface-container-low` (#131314) panel to create a recessed, "carved" look. 
*   **Ambient Shadows:** If an element must float (like a dropdown), use a shadow with a blur of 40px, 0% spread, and an opacity of 6% using the `on-surface` color. This mimics natural light rather than a digital effect.
*   **The "Ghost Border":** For high-density data where tonal shifts aren't enough, use a border with the `outline-variant` (#484849) token set to **15% opacity**. It should be felt, not seen.
*   **Glassmorphism:** Use semi-transparent layers for non-modal overlays to allow the `tertiary` (success green) or `primary` (blue/purple) colors to bleed through from the content below, maintaining a sense of spatial awareness.

## 5. Components
All components follow a strict **Roundedness Scale**. Use `DEFAULT` (0.5rem) for inputs and `lg` (1rem) for containers to maintain a "soft-tech" feel.

*   **Buttons:**
    *   *Primary:* Background is the primary-to-secondary gradient. Text is `on-primary-fixed` (#000000).
    *   *Secondary:* `surface-container-high` (#201f21) with a "Ghost Border."
*   **Cards:** Forbid divider lines. Use `surface-container-lowest` and separate content using the `xl` (1.5rem) spacing scale.
*   **Input Fields:** Use `surface-container-highest` (#262627) for the field background. The active state should not use a thick border, but a 1px `primary-dim` (#6063ee) "Ghost Border" and a subtle outer glow.
*   **Invoicing Shields:** Incorporate custom "Secure" badges using the `tertiary` (#c5ffc9) success green and the line-style shield icons. These should be small, placed near "Save" or "Send" actions to reinforce compliance and trust.
*   **Chips:** Use `surface-variant` (#262627) for inactive states and the `tertiary-container` (#6bff8f) for "Paid" or "Validated" states.

## 6. Do's and Don'ts

### Do:
*   **DO** use whitespace as a separator. If you feel the need for a line, increase the padding by 8px instead.
*   **DO** use the `tertiary` green (#c5ffc9) only for validation. It is the color of "Money In" and "Legal Compliance."
*   **DO** keep typography high-contrast. Use `on-surface` for headings and `on-surface-variant` for secondary labels to guide the eye.

### Don't:
*   **DON'T** use pure #000000 for backgrounds unless it is a recessed "lowest" container. Use the `surface` (#0e0e0f) for the main canvas.
*   **DON'T** use high-opacity borders. They create "visual noise" that fatigues the user during long accounting sessions.
*   **DON'T** use standard blue for links. Use the `primary` (#a3a6ff) or `secondary` (#c180ff) tones to maintain the bespoke Moroccan SaaS identity.
*   **DON'T** crowd the logo. The GEKKO brand represents agility; give the logo at least 40px of "breathing room" (safe zone) in any layout.

***

*Director's Note: This system is about the balance between the heavy responsibility of ERP and the lightness of modern SaaS. When in doubt, simplify the container and amplify the typography.*```