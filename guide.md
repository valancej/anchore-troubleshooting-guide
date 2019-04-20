

- services overview (which services need what to operate)
- system commands via the cli (API too)
    - system status should return on success
    - location of logs in each container
    - which service logs to check depending on: 
        - No feed data
        - Failed analysis
            - analyzer log
        - Failed policy eval
            - catalog log
            - policy log
        - Image analysis process in general
        - CLI commands in detail
            - system status
            - system feeds list
            - events list
            - target specific events
            - --debug and --json flags
            - Forcing a sync and flushing the db
        - Failure to add registry
            - --debug flag
        - Long image wait times
            - analyzer service log
            - performance tuning docs and pre-deployment sizing considerations

        