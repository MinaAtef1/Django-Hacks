---
layout: default
title: Create admin registry class to keep track of all admin instances
nav_order: 2
parent: Django Admin
---

## Create Admin Registry Class to Keep Track of All Admin Instances

After creating multiple admin instances, we should create some kind of register to keep track of all admin sites created and to register the urls of each admin site.

```python
from django.urls import path


# A singleton class to register admin sites
class AdminRegister(object):
    admin_sites = {}

    def __new__(self):
        if not hasattr(self, 'instance'):
            self.instance = super(AdminRegister, self).__new__(self)
        return self.instance
        

    def register(self, admin_site, icon,name,title,url)->None:
        self.admin_sites[admin_site] = {
            'icon': icon,
            'name': name,
            'title': title,
            'url': url,
            'app': f'{admin_site.name}:index',
            'app_list_function': admin_site.get_app_list,
        }

    def get_url_paths(self)->list:
        paths = []
        for site, site_dict in self.admin_sites.items():
            paths.append(path(site_dict['url'], site.urls))
        return paths

    def get_admin_sites(self)->dict:
        return self.admin_sites

    def get_admin_sites(self, request)->list:
        data = []
        for site, site_dict in self.admin_sites.copy().items():
            if site.has_permission(request):
                site_dict['apps_list'] = site_dict['app_list_function'](request) if request else None
                data.append(site_dict)

        return data

```

so with this let's register some admin sites
    
```python
class MyAdminAdmin(ModelsAdmin):
    site_header = "My Admin"
    site_title = "My Admin site"
    index_title = "My Admin"

my_admin = MyAdminAdmin(name='my_admin')
AdminRegister().register(assertion_admin, 'fa fa-admin', 'my_admin', 'my_admin', 'my_admin/')
```


now that we have an registered all the admin let's add the urls
```python
urlpatterns = [
    ...
] + AdminRegister().get_url_paths()
```
