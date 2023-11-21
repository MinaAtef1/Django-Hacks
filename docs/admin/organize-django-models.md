---
layout: default
title: organize django models
nav_order: 3
parent: Django Admin
---

## organize django models

When working with multiple admin sites in Django, there might arise a need to re-organize the admin list for better clarity and ease of navigation. In this article, we'll explore how to organize Django models in a hierarchical structure within the admin list.

### The Need for Organization

Consider a scenario where your Django app consists of various models, such as

- orders
- products
- suppliers
- payments
- invoices

To enhance the admin user experience, you might want to group related models together, creating a more intuitive and structured view.

- orders models
  - orders
  - invoices
- products models
  - products
  - suppliers
- payments models
  - payments


To achieve this, we'll create a custom `ModelsAdmin` class that inherits from the default `AdminSite` in Django. This class allows us to override key methods responsible for generating the app dictionary and organizing the models.


```python

class ModelsAdmin(admin.AdminSite):
    def __init__(self, *args, **kwargs):
        self.sub_apps = {}
        super().__init__(*args, **kwargs)

    def get_sub_models(self):
        """
        Get a dictionary of sub-models, mapping each model to its corresponding sub-app verbose name.
        """
        models_sub = {}
        for sub_app_name, sub_app_dict in self.sub_apps.items():
            for model, model_dict in sub_app_dict['models'].items():
                models_sub[model] = sub_app_dict['sub_app_verbose_name']
        return models_sub

    def generate_app_dict(self, request):
        """
        This ia a native django method that we override to organize the admin list
        """

        def get_init_app_dict(app_name):
            return {
                'name': app_name,
                'models': [],
                'has_module_perms': True,
                'models_url_list': []
            }.copy()

        app_dict = {}
        config_dict = self._build_app_dict(request)
        sup_models = self.get_sub_models()

        for app_label, app_config in config_dict.items():
            for model in app_config['models']:

                models_class = model['model']
                app_verbose = sup_models.get(models_class, app_label)

                app_dict.setdefault(app_verbose, get_init_app_dict(app_verbose))
                app_dict[app_verbose]['models_url_list'].append(model['admin_url'])
                app_dict[app_verbose]['models'].append(model)
        return app_dict


    def get_model_sort(self, model_class):
        """
        Get the sort order of a model.
        """
        models_order = {}
        for sub_app_name, sub_app_dict in self.sub_apps.items():
            for model, model_dict in sub_app_dict['models'].items():
                models_order[model] = model_dict['model_order']
        return models_order.get(model_class, float('inf'))


    def get_app_list(self, request, app_label=None):
        """
        Return a sorted list of all the installed apps that have been
        registered in this site.
        """
        app_dict = self.generate_app_dict(request)

        # Sort the apps alphabetically.
        app_list = sorted(app_dict.values(), key=lambda x: self.sub_apps.get(x['name'], x['name']))
        # Sort the models alphabetically within each app.
        for app in app_list:
            app['models'].sort(key=lambda x: self.get_model_sort(x.get('model', None)))

        return app_list

    def register_sub_app(self, sub_app_name, sub_app_order):
        """
        Register a sub-app in the admin and return a function to register models under this sub-app.
        """
        self.sub_apps[sub_app_name] = {'sub_app_name': sub_app_name,
                                       'sub_app_verbose_name': sub_app_name.replace('_', ' ').title(),
                                       'sub_app_order': sub_app_order,
                                       'models': {}}

        def sub_app_register(model_or_iterable, admin_class=None, model_order=None, **options):
            self.register(model_or_iterable, admin_class, **options)
            self.sub_apps[sub_app_name]['models'][model_or_iterable] = {
                'model_order': model_order}

        return sub_app_register

```

now we can register any sub app in the admin like this 
    
```python
orders_sub_app_register = my_admin_site.register_sub_app('orders',0)
# then use it to register my models 
orders_sub_app_register(OrdersModel, OrdersAdmin, model_order=0)

```