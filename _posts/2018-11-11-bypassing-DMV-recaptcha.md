Booking a DMV appointment in the bay area is very inconvenient as one has to examine all candidate DMV offices one by one. I spent some time during the weekend about how we can batch check the DMV appointment availability. This post discusses my findings.

## Recaptcha Protection
[DMV appointment submission form](https://www.dmv.ca.gov/wasapp/foa/findOfficeVisit.do) is not behind the login wall. Thus, they used Google's [Recaptcha](https://www.google.com/recaptcha/intro/v3.html#the-recaptcha-advantage) to protect the page from bots. *Recaptcha verification is a two-phase process. First, in frontend the webpage talks to Recaptcha service and get a response. Then, the response is submitted by user together with other fields to the DMV server side, and the server connects with Recaptcha service again on the response validity.* Our first trial using selenium failed as recaptcha blocked us from moving on.

```python
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import Select
import time

def webserver_approach():
    driver = webdriver.Firefox()
    driver.get("https://www.dmv.ca.gov/wasapp/foa/clear.do?goTo=officeVisit")
    for x in driver.find_elements_by_xpath('.//a'):
        if x.get_attribute('text').strip() == "Clear Fields":
            x.click()
            break
    office_elem = driver.find_element_by_id("officeId")
    Select(office_elem).select_by_visible_text("SAN FRANCISCO")
    driver.find_element_by_id("taskRID").click()
    driver.find_element_by_id("first_name").send_keys("first")
    driver.find_element_by_id("last_name").send_keys("last")
    driver.find_element_by_id("areaCode").send_keys("650")
    driver.find_element_by_id("telPrefix").send_keys("123")
    driver.find_element_by_id("telSuffix").send_keys("4567")
    driver.find_element_by_id("telSuffix").send_keys(Keys.RETURN)
    driver.close()
```

Unsurprisingly, this code was blocked by Recaptcha and we could not move further.

## Trick Recaptcha Response
Looking closer at successful appointment submissions in Chrome Dev Tools, it has two fields *captchaResponse* and *g-recaptcha-response*. On the server side, the server sent this response to Google and asked whether it was a human or not. We can then trick the DMV website by providing a valid recaptcha response.

```python
def clean_text(text):
    return '\t'.join([x for x in re.split('\r|\t|\n', text) if x])


def dmv_request(session=None, officeId="599", successful_response=None):
    headers = {'User-Agent': 'Mozilla/5.0'}
    payload = {
        "mode": "OfficeVisit",
        "captchaResponse": successful_response,
        "officeId": officeId,
        "numberItems": "1",
        "taskRID": "true",
        "firstName": "JACK",
        "lastName": "DOE",
        "telArea": "650",
        "telPrefix": "111",
        "telSuffix": "1111",
        "resetCheckFields": "true",
        "g-recaptcha-response": successful_response,
    }

    if not session:
        session = requests.Session()
    res = session.post('https://www.dmv.ca.gov/wasapp/foa/findOfficeVisit.do', headers=headers, data=payload)
    soup = BeautifulSoup(res.text, 'html.parser')
    table = soup.select('#formId_1 > div > div.r-table.col-xs-12 > table > tbody > tr')
    return f"{clean_text(table[0].find('td', {'data-title': 'Office'}).text)}\n{clean_text(table[0].find('td', {'data-title': 'Appointment'}).text)}"
```

This works well if you copy the recaptcha response from a successful request. If you fail to provide one or provide a wrong one, the request would fail. Now let's get all the available slots in the bay area. You can also modify the script a bit for periodical monitoring.

```python
import requests
import time

CITIES = {
    "FREMONT": 644,
    "HAYWARD": 579,
    "LOS GATOS": 640,
    "REDWOOD CITY": 548,
    "SAN FRANCISCO": 503,
    "SAN JOSE": 516,
    "SAN MATEO": 593,
    "SANTA CLARA": 632,
    'DALY CITY': 599,
}

session = requests.Session()
for city, city_id in CITIES.items():
    print(dmv_request(session, str(city_id)))
    print('---------------------------')
    time.sleep(2)
```

Finally, where do we find all the office IDs in the bay area? Here is a simple JS solution. An even simpler way is going to the [page source](https://www.dmv.ca.gov/wasapp/foa/clear.do?goTo=officeVisit) and copy the dropdown fields.

```javascript
Array.from(document.getElementById("officeId").options).map(x => { return [x.text, x.value] });
```

## Summary
Although recaptcha looks intimating, there can be holes in the implementations and it is up to us to find out :)