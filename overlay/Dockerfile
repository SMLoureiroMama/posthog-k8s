# PostHog's web image plus the trusted-header login (docs/auth-proxy.md). Nothing of PostHog is
# replaced: one module is added and posthog/settings/__init__.py gets a hook at its end that does
# nothing unless AUTH_PROXY_ENABLED is set.
ARG POSTHOG_IMAGE
FROM ${POSTHOG_IMAGE}

COPY --chown=posthog:posthog posthog_k8s_authproxy /code/posthog_k8s_authproxy

# The greps fail the build when upstream moves what the login is anchored to. That the module
# loads in this PostHog is checked where web starts with it: the end-to-end run.
RUN set -eu; \
    cd /code; \
    grep -q '"posthog.middleware.AuthenticationMiddleware"' posthog/settings/web.py; \
    grep -q 'def get_original_user_from_session' posthog/helpers/impersonation.py; \
    python -m compileall -q posthog_k8s_authproxy; \
    printf '\n# posthog-k8s: trusted-header login, inert without AUTH_PROXY_ENABLED\nfrom posthog_k8s_authproxy.settings import apply as _auth_proxy_apply  # noqa: E402\n\n_auth_proxy_apply(globals())\n' \
      >> posthog/settings/__init__.py
