# servicenow_scriptincludesampleBETTER
servicenow_scriptincludesampleBETTER


Here is the consolidated, revised implementation. It incorporates the feedback while preserving the correct GlideAjax pattern and efficient server-side aggregation.

## 1. Client Script — onChange

Configuration: Table = `Incident`, Field name = `Assigned to`, Type = `onChange`.

```
function onChange(control, oldValue, newValue, isLoading, isTemplate) {
    if (isLoading || isTemplate || !newValue) {
        return;
    }

    var threshold = 10;
    var fieldName = control.name;

    var ga = new GlideAjax('IncidentUtils');
    ga.addParam('sysparm_name', 'getOpenIncidentCount');
    ga.addParam('sysparm_assignee', newValue);

    ga.getXMLAnswer(function(answer) {

        // Ignore the response if the field has changed
        if (g_form.getValue(fieldName) !== newValue) {
            return;
        }

        var count = parseInt(answer, 10);

        // Ignore invalid responses
        if (isNaN(count) || count < 0) {
            return;
        }

        // Clear previous form messages before displaying a new warning
        g_form.clearMessages();

        if (count > threshold) {
            var assigneeName = g_form.getDisplayValue(fieldName);

            g_form.addInfoMessage(
                assigneeName + ' currently has ' + count +
                ' active incidents assigned (threshold: ' +
                threshold + '). Please consider the workload.'
            );
        }
    });
}
```

## 2. Script Include — `IncidentUtils`

Configuration: Active = true; Client callable = true; Accessible from the calling application. Use the appropriate API name if the Script Include is scoped.

```
var IncidentUtils = Class.create();

IncidentUtils.prototype = Object.extendsObject(
    global.AbstractAjaxProcessor, {

        getOpenIncidentCount: function() {
            var assignee = this.getParameter('sysparm_assignee');

            // Validate the supplied user sys_id
            if (!assignee ||
                !/^[0-9a-f]{32}$/i.test(assignee)) {
                return '-1';
            }

            var ga = new GlideAggregate('incident');
            ga.addQuery('assigned_to', assignee);
            ga.addQuery('active', true);
            ga.addAggregate('COUNT');
            ga.query();

            var count = 0;

            if (ga.next()) {
                count = parseInt(
                    ga.getAggregate('COUNT'), 10
                ) || 0;
            }

            return count.toString();
        },

        type: 'IncidentUtils'
    }
);
```

Scope adjustment: The code above uses `global.AbstractAjaxProcessor`, appropriate for a scoped application. If `IncidentUtils` is in the global scope, use `AbstractAjaxProcessor` instead. Ensure the client-side GlideAjax name matches the Script Include's actual API name.

## 3. Important implementation notes

- `isTemplate` guard: Included to skip execution during template application.
- `!newValue`: Handles empty values and other falsy values.
- `control.name`: Avoids hard-coding `assigned_to` in the stale-response check.
- `g_form.clearMessages()`: Prevents old form messages from accumulating, but it clears all form messages, not just this warning. If other messages must remain visible, use a more targeted message-management strategy.
- Invalid response: The server returns `'-1'` for invalid input, and the client ignores negative counts.
- Open versus active: The query currently counts all incidents with `active = true`. Change the query if your business defines “open” by specific states instead.
- Security: Client-callable does not mean unrestricted access should be allowed. Apply appropriate authorization if workload counts are sensitive.

One final distinction: this implementation warns the user when the count exceeds 10; it does not prevent the assignment. A hard enforcement requirement needs server-side validation as well.
