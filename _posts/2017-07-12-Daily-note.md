

- Appium python sample

```python
import os
import appium.webdriver

webdriver = appium.webdriver

# Returns abs path relative to this file and not cwd
PATH = lambda p: os.path.abspath(
    os.path.join(os.path.dirname(__file__), p)
)

desired_caps = {}
desired_caps['deviceName'] = 'TestApp'
desired_caps['platformName'] = 'Android'
# desired_caps['browserName'] = 'browser'
#desired_caps['version'] = '5.0.2'
desired_caps['app'] = PATH('/Users/ml/dev/appium/ContactManager.apk')
desired_caps['app-package'] = 'com.example.android.contactmanager'
desired_caps['app-activity'] = '.ContactManager'
desired_caps["platformVersion"] = "5.0.2"
desired_caps["noReset"] = True
#desired_caps["chromeOptions"] = {"args": ['--disable-popup-blocking']}

driver = webdriver.Remote('http://localhost:4723/wd/hub', desired_caps)

el = driver.find_element_by_id("Add Contact")
el.click()

textfields = driver.find_element_by_id("com.example.android.contactmanager:id/contactNameEditText")
print('textfileds: ')
print(textfields)
print('\n')
print('its methods: ')
print(dir(textfields))
textfields.send_keys(["My Name"])
#textfields[2].send_keys("someone@somewhere.com")

driver.find_element_by_id("Save").click()

driver.quit()
```
