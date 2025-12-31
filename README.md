# Project Documentation

## claim Command

The `claim` command allows users to claim rewards under certain conditions. Prior versions of this command had inconsistencies in handling user outcomes, and cooldowns were not consistently applied.

### Updates to `claim`

#### All Outcome Responses:
As of the latest implementation, the `claim` command ensures that responses are sent for all possible outcomes:
- Success: The user receives the reward and confirmation message.
- Failure: Informative error messages are provided for issues such as ineligibility or claim restrictions.

#### Cooldown Management:
The `claim` command now has proper handling of cooldown periods to prevent multiple claims within restricted intervals. The system will:
- Deny the claim request during an active cooldown period.
- Inform users about the remaining cooldown time in a clear and precise response.

Ensure to test the `claim` command for edge cases to validate these improvements. For contribution or further issues, please submit a pull request or open an issue.