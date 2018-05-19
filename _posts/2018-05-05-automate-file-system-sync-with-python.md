My day-to-day work involves a remote development server. From time to time I need to keep my local working directory in sync with remote development server with `rsync`. Also, for security reasons, I am required to manually input my credentials to authorize the sync every few hours. It is really annoying! I thus wrote a simple tool to automate this process. It used [watchdog](https://pythonhosted.org/watchdog/) to monitor local file system changes, and getpass/[pexpect](https://pexpect.readthedocs.io/en/stable/) for feeding my password to the auth program.

## Code Walk-through
We assume the working directory to monitor is set in env variable `DATA_DIR`. [watchdog](https://pythonhosted.org/watchdog/) can then monitor the directory file changes and set `NEED_RESYNC` flag to True. Here we define a handler `FileChangeHandler` that captures all change events including *created*, *modified*, *deleted*, *moved*. Every time these events happen. In the main thread, we loop infinitely and `rsync` the file systems when `NEED_RESYNC` is true.

Alternatively, you can initiate the rsync command under every file change. However, this can possibly cause multiple rsync commands running at the same time. With `NEED_RESYNC` flag, we can serialize rsync command and control the sync frequency.

```python
import subprocess
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import getpass
import pexpect

class Watcher:
    NEED_RESYNC = False

    def __init__(self):
        self.path = os.environ['DATA_DIR']

        class FileChangeHandler(FileSystemEventHandler):
            @staticmethod
            def on_any_event(event):
                if event.is_directory:
                    return None
                elif event.event_type in ('created',
                    'modified', 'deleted', 'moved'):
                    logger.info("Received {} event - {}. re-sync".format(
                        event.event_type, event.src_path))
                    Watcher.NEED_RESYNC = True

        self.event_handler = FileChangeHandler()

    def run(self):
        observer = Observer()
        observer.schedule(self.event_handler, self.path, recursive=True)
        observer.start()
        try:
            while True:
                if Watcher.NEED_RESYNC:
                    try:
                        subprocess.check_output(['your_rsync_command'])
                        Watcher.NEED_RESYNC = False
                    except Exception as e:
                        logger.error('seeing auth exception!')
                        logger.exception(e)
                        self.manual_auth()
                time.sleep(2)
        except KeyboardInterrupt:
            observer.stop()
        observer.join()
```

I do not want to skip the authorization process. So I sent a prompt to wait for user credentials with getpass, and feed the credentials to the auth command with pexpect. 

```python
class Watcher:
    # ...
    def manual_auth(self):
        pw = getpass.getpass('Input your password:')
        yk = pexpect.spawn('your_auth_command')
        yk.logfile_read = sys.stdout
        yk.expect('auth_command_prompt')
        yk.sendline(pw)
        yk.expect('auth_success_confirmation')
        yk.wait()
```

I am always impressed by what Python can help us in our everyday work. Hope you also love it!
