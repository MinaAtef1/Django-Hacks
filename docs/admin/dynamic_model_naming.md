---
layout: default
title: Dynamic Model Naming
nav_order: 4
parent: Django Admin
---

## Dynamic Model Naming

After creating our custom `ModelsAdmin` class, and our own `register_sub_app` function, we might need to add a some way to change the model name and maybe the color

to do so, we will modify our register function to accept a naming function as follows:

```python
    def register_sub_app(self, sub_app_name, sub_app_order):
        """
        Register a sub-app in the admin and return a function to register models under this sub-app.
        """
        self.sub_apps[sub_app_name] = {'sub_app_name': sub_app_name,
                                       'sub_app_verbose_name': sub_app_name.replace('_', ' ').title(),
                                       'sub_app_order': sub_app_order,
                                       'models': {}}

        def sub_app_register(model_or_iterable, admin_class=None, model_order=None,naming_function=None, **options):
            self.register(model_or_iterable, admin_class, **options)
            self.sub_apps[sub_app_name]['models'][model_or_iterable] = {'model_order': model_order,
                                                                    'naming_function': naming_function}
        return sub_app_register
```


and then we will modify the `generate_app_dict` function and start using the naming function

```python
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

                # we added this part
                naming_function = self.sub_apps.get(app_label, {}).get('models', {}).get(models_class, {}).get('naming_function', None)
                if naming_function:
                    model['name'] = naming_function()

                app_dict[app_verbose]['models'].append(model)

        return app_dict
```

now we could use the naming function to change the model name, for example:

```python
def orders_admin_name():
    draft_orders = Order.objects.filter(status=Order.OrderStatus.DRAFT)
    if draft_orders:
        return 'Orders (' + str(draft_orders.count()) + ')'
    return 'Orders'

orders_admin_register(Order, OrderAdmin,naming_function=orders_admin_name)
```