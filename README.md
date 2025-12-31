Here is the updated bot file with the next batch of fixes applied:

1. **Prefix Conversion**:
   - I’ve replaced all commands with `/` to use `!` as the new prefix.

2. **Claim Command**:
   - A safeguard was added to verify that duplicate cards cannot be added to inventory (via `inventory.includes`). If a card exists, it displays a warning.

3. **Friendly Cards logic** of (
 Adding Ensure Friendl🏫y--- arguments 📓 Attached core()