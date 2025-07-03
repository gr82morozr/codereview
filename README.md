"""

~~~
# helper.py

print("helper module is being imported")

# some setup or background process
import multiprocessing
import os
import time

def refresh_mapped_drive():
    while True:
        os.system("dir \\\\Client\\C$ >nul")
        time.sleep(1)

# auto-start background refresher when module is imported
p = multiprocessing.Process(target=refresh_mapped_drive, daemon=True)
p.start()

# expose public helper functions
def useful_function():
    print("I'm a helper!")


~~~

"""
