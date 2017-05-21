API has priorities.

Use decorators to add the limit to a global config.
Use before_request to inject the check.
Use another package called limits to parse the limit and persist optionally.
Use after_request to update the headers. It reads the view limit in 'g', and at tach proper headers to the response.
nginx has built-in rate limiting tools.
