Twitch
======

Authentication
--------------

**In order to get the personal OAuth token from Twitch's website which identifies your account**, open Twitch.tv in your web
browser and after a successful login, open the developer tools by pressing :kbd:`F12` or :kbd:`CTRL+SHIFT+I`. Then navigate to
the "Console" tab or its equivalent of your web browser and execute the following JavaScript snippet, which reads the value of
the ``auth-token`` cookie, if it exists:

.. code-block:: javascript

    document.cookie.split("; ").find(item=>item.startsWith("auth-token="))?.split("=")[1]

Copy the resulting string consisting of 30 alphanumerical characters without any quotations.

The final ``Authorization`` header which will identify your account while requesting a streaming access token can then be set
via Streamlink's :option:`--twitch-api-header` or :option:`--http-header` CLI arguments.

The value of the ``Authorization`` header must be in the format of ``OAuth YOUR_TOKEN``. Notice the space character in the
argument value, which requires quotation on command line shells:

.. code-block:: console

    $ streamlink "--twitch-api-header=Authorization=OAuth abcdefghijklmnopqrstuvwxyz0123" twitch.tv/CHANNEL best

The entire argument can optionally be added to Streamlink's (Twitch plugin specific)
:ref:`config file <cli/config:Plugin specific configuration file>`, which :ref:`doesn't require quotes <cli/config:Syntax>`:

.. code-block:: text

    twitch-api-header=Authorization=OAuth abcdefghijklmnopqrstuvwxyz0123

.. dropdown:: Changelog
    :animate: fade-in-slide-down
    :chevron: right-down
    :icon: log

    Official authentication support for Twitch via the ``--twitch-oauth-token`` and ``--twitch-oauth-authenticate``
    CLI arguments had to be disabled in :ref:`streamlink 1.3.0 <changelog:streamlink 1.3.0 (2019-11-22)>` (November 2019)
    and both arguments were finally removed in :ref:`streamlink 2.0.0 <changelog:streamlink 2.0.0 (2020-12-22)>` (December 2020)
    due to `restrictive changes`_ on Twitch's private REST API which prevented proper authentication flows
    from third party applications like Streamlink.

    The issue was that authentication data generated from third party applications could not be sent while acquiring
    streaming access tokens which are required for watching streams. Only authentication data generated by Twitch's website
    was accepted by the Twitch API. Later on in January 2021, Twitch moved the respective API endpoints to their GraphQL API
    which was already in use by their website for several years and shut down the old, private REST API.

    This means that authentication data, aka. the "OAuth token", needs to be read from the web browser after logging in
    on Twitch's website and it then needs to be set as a certain request header on these API endpoints. This unfortunately
    can't be automated easily by applications like Streamlink, so a new authentication feature was never implemented.

.. _restrictive changes: https://github.com/streamlink/streamlink/issues/2680#issuecomment-557605851


Embedded ads
------------

**Streamlink will automatically filter out preroll and midroll ads on Twitch.**
A message with the expected advertisement time will be logged.

Filtering out ads means that the stream output will be paused during that time, and the player will keep trying to read data
until the real stream becomes available again. This also means that there will always be a stream discontinuity
when the output resumes, aka. a gap of the cohesively encoded stream data. This is expected and can't be circumvented.
Most :ref:`players <players:Players>` should be able to recover from this kind of discontinuity though,
unlike when concatenating data of the real stream and ads.

Completely preventing ads may be possible by :ref:`authenticating <cli/plugins/twitch:Authentication>` (Twitch Turbo)
or via special Twitch API request headers and/or parameters that modify the access token acquirement, if the community is aware
of such loop-holes. See :option:`--twitch-api-header` and :option:`--twitch-access-token-param`.

.. dropdown:: Changelog
    :animate: fade-in-slide-down
    :chevron: right-down
    :icon: log

    In 2019, Twitch started embedding ads directly into streams in addition to their regular advertisement program
    on their website which can only overlay ads. While this may be an annoyance for people who are used to using ad-blocker
    extensions in their web-browsers for blocking regular overlaying ads, applications like Streamlink face another problem,
    namely stream discontinuities when there's a transition between the regular stream content and ad segments.

    Since Streamlink does only output a single progressive stream from reading Twitch's segmented HLS stream,
    ads can cause playback issues, as the output is not a cohesively encoded stream of audio and video data anymore during
    an ad transition. One of the problematic players is :ref:`VLC <players:Players>`, which is known to crash during these
    stream discontinuities in certain cases.

    Prior releases between :ref:`streamlink 1.1.0 <changelog:streamlink 1.1.0 (2019-03-31)>` (March 2019)
    and :ref:`streamlink 7.5.0 <changelog:streamlink 7.5.0 (2025-07-08)>` (July 2025) required the ``--twitch-disable-ads``
    plugin argument, as filtering out ads was deemed optional. Ad filtering became mandatory when Twitch changed the stream's
    format from MPEG-TS to MPEG-4, to prevent playback issues during stream discontinuities between the stream and ads.


Client-integrity token
----------------------

Sometimes when acquiring a streaming access token from Twitch for watching streams, a client-integrity token might be required.
CI tokens are supposed to prove the legitimacy of the user and thus filter out bots. They are calculated using sophisticated
JavaScript code in the user's web browser that's infeasible to reverse engineer or translate to Python.

When such a CI token is required, or if the user has set the :option:`--twitch-force-client-integrity` argument,
the Twitch plugin will use Streamlink's :ref:`streamlink.webbrowser <api/webbrowser:Webbrowser>` API, which requires
a Chromium-based web browser to be installed on the user's system, so that the CI token can be calculated.
See the :option:`--webbrowser` and related CLI arguments for more details.

If supported by the Chromium-based web browser and the environment Streamlink is run in, :option:`--webbrowser-headless`
allows hiding the web browser's window.

CI tokens will be cached for as long as they are valid, to prevent having to launch the local web browser every time.
:option:`--twitch-purge-client-integrity` allows clearing the cached token.

.. dropdown:: Changelog
    :animate: fade-in-slide-down
    :chevron: right-down
    :icon: log

    In 2022, Twitch added client-integrity tokens to their web player when getting streaming access tokens.
    CI tokens were treated as an optional request parameter when getting streaming access tokens, but this changed
    at the end of May in 2023 when Twitch made them a requirement for a week, which broke Streamlink's Twitch plugin (#5370).

    Since the only sensible solution for Streamlink to calculate CI tokens was using a web browser,
    the :ref:`streamlink.webbrowser <api/webbrowser:Webbrowser>` API was implemented in
    :ref:`streamlink 6.0.0 <changelog:streamlink 6.0.0 (2023-07-20)>` (July 2023).


Low latency streaming
---------------------

Low latency streaming on Twitch can be enabled by setting the :option:`--twitch-low-latency` argument and (optionally)
configuring the :ref:`player <players:Players>` via :option:`--player-args` and reducing its own buffer to a bare minimum.

Setting :option:`--twitch-low-latency` will make Streamlink prefetch future HLS segments that are included in the HLS playlist
and which can be requested ahead of time. As soon as content becomes available, Streamlink can download it without having to
waste time on waiting for another HLS playlist refresh that might include new segments.

In addition to that, :option:`--twitch-low-latency` also reduces :option:`--hls-live-edge` to a value of at most ``2``, and it
also sets the :option:`--hls-segment-stream-data` argument.

:option:`--hls-live-edge` defines how many HLS segments Streamlink should stay behind the stream's live edge, so that it can
refresh playlists and download segments in time without causing buffering. Setting the value to ``1`` is not advised due to how
prefetching works.

:option:`--hls-segment-stream-data` lets Streamlink write the content of in-progress segment downloads to the output buffer
instead waiting for the entire segment to complete first before data gets written. Since HLS segments on Twitch have a playback
duration of 2 seconds for most streams, this further reduces output delay.

.. note::

    Low latency streams have to be enabled by the broadcasters on Twitch themselves. Regular streams can cause buffering issues
    with this option enabled due to the reduced :option:`--hls-live-edge` value.

    Unfortunately, there is no way to check whether a channel is streaming in low-latency mode before accessing the stream.

Player buffer tweaks
^^^^^^^^^^^^^^^^^^^^

Since players do have their own input buffer, depending on how much data the player wants to keep in its buffer before it starts
playing the stream, this can cause an unnecessary delay while trying to watch low latency streams. Player buffer sizes should
therefore be tweaked via the :option:`--player-args` CLI argument or via the player's configuration options.

The delay introduced by the player depends on the stream's bitrate and how much data is necessary to allow for a smooth playback
without causing any stuttering, e.g. when running out out available data.

Please refer to the player's own documentation for the available options.
