# Third-Party Service Integrations

A developer's guide for Specter Desktop `Service` integrations.


## Basic Code Philosophy
As much as possible, each `Service` implementation should be entirely self-contained with little or no custom code altering existing/core Specter functionality.

Effort has been taken to provide `Service` data storage that is separate from existing data stores in order to keep those areas clean and simple. Where touchpoints are unavoidable, they are kept to the absolute bare minimum (e.g. `User.services` list, `Address.service_id` field).


## `Address`-Level Integration
An `Address` can be associated with a `Service` (e.g. addr X received a smash buy from `Service` Foo) via the `Address.service_id` field.

A `Service` can also "reserve" and `Address` for future use by setting `Address.service_id`. The normal "Receive" UI will automatically skip any reserved `Address`es when generating a new receive addr. The reserved addresses are interleaved with ready-to-use addresses so that we don't create any potentially confusing wallet gaps (e.g. addrs 4, 6, and 8 are reserved but addrs 3, 5, and 7 are available).

Users can also manually associate an existing `Address` with a `Service` (this is useful when the user has info that the particular `Service` api can't provide for whatever reason).

_Note: TODO: manually un-reserve an `Address` from a `Service`._


## Basic Code Structure
All `Service`-related code should be contained within `cryptoadvance.specter.services`. The base components are:


### `Service` Base Class
Defines the base `Service` class that all service integrations must inherit from. This is wired to enable `Service` auto-discovery. Any feature that is common to most or all `Service` integrations should be implemented here.

Each `Service` must specify a unique `Service.id` that is just a short string (e.g. "swan"). This is the main identifier throughout the code.

Includes methods to "reserve" addresses for the `Service` to basically make those not-yet-used addresses somewhat off-limits to the rest of the UI (can still be manually overridden though).


### `Service` Configuration
In order to separate the service-configuration from the main-configuration, you can specify your config in a file called `config.py`. It's structure is similiar to the specter-wide `config.py`, e.g.:
```
class BaseConfig():
    SWAN_API_URL="https://dev-api.swanbitcoin.com"

class ProductionConfig(BaseConfig):
    SWAN_API_URL="https://api.swanbitcoin.com"
```
In your code, you can access the correct value as in any other flask-code, like `api_url = app.config.get("SWAN_API_URL")`. If the instance is running a config (e.g. `DevelopmentConfig`) which is not available in your service-specific config (as above), the inheritance-hirarchy from the mainconfig will get traversed and the first hit will get get configured. In this example, it would be `BaseConfig`.


### `ServiceManager`
Simple manager that contains all `Service`s. Performs the `Service` auto-discovery at startup and filters availability by each `Service`'s release level (i.e. alpha, beta, etc).


### `ServiceEncryptedStorage`
Most `Service`s will require user secrets (e.g. API key and secret). Each Specter `User` will have their own on-disk encrypted `ServiceEncryptedStorage` with filename `<username>_services.json`. Note that the user's secrets for all `Service`s will be stored in this one file.

This is built upon the `GenericDataManager` class which supports optional encrypted fields. In this case all fields are encrypted. The `GenericDataManager` encryption can only be unlocked by each `User`'s individual `user_secret` that itself is stored encrypted on-disk; it is decrypted to memory when the `User` logs in.

For this reason `Service`s cannot be activated unless the user is signing in with a password-protected account (the default no-password `admin` account will not work).

_Note: during development if the Flask server is restarted or auto-reloads, the user's decrypted `user_secret` will no longer be in memory. The Flask context will still consider the user logged in after restart, but code that relies on having access to the `ServiceEncryptedStorage` will throw an error and/or prompt the user to log in again._

It is up to each `Service` implementation to decide what data is stored; the `ServiceEncryptedStorage` simply takes arbitrary json in and delivers it back out.

This is also where `Service`-wide configuration or other information should be stored, _**even if it is not secret**_ (see above intro about not polluting other existing data stores).


### `ServiceEncryptedStorageManager`
Because the `ServiceEncryptedStorage` is specific to each individual user, this manager provides convenient access to automatically retrieve the `current_user` from the Flask context and provide the correct user's `ServiceEncryptedStorage`. It is implemented as a `Singleton` which can be retrieved simply by importing the class and calling `get_instance()`.

This simplifies code to just asking for:
```
from .service_encrypted_storage import ServiceEncryptedStorageManager

ServiceEncryptedStorageManager.get_instance().get_current_user_service_data(service_id=some_service_id)
```

As a further convenience, the `Service` base class itself encapsulates `Service`-aware access to this per-`User` encrypted storage:
```
@classmethod
def get_current_user_service_data(cls) -> dict:
    return ServiceEncryptedStorageManager.get_instance().get_current_user_service_data(service_id=cls.id)
```

Whenever possible, external code should not directly access these `Service`-related support classes but rather should ask for them through the `Service` class.


### `ServiceAnnotationsStorage`
Annotations are any address-specific or transaction-specific data from a `Service` that we might want to present to the user. Example: a `Service` that integrates with a onchain storefront would have product/order data associated with a utxo. That additional data could be imported by the `Service` and stored as an annotation. This annotation data could then be displayed to the user when viewing the details for that particular address or tx.

Annotations are stored on a per-wallet and per-`Service` basis as _unencrypted_ on-disk data (filename: `<wallet_alias>_<service>.json`).

_Note: current `Service` implementations have not yet needed this feature so displaying annotations is not yet implemented._


### `controller.py`
The minimal url routes for `Service` selection and management.


## Implementation Class Structure
Child implementation classes (e.g. `SwanService`) should be self-contained within their own subdirectory in `services`. e.g.:
```
cryptoadvance.specter.services.swan
```

Each implementation must have the following required components:
```
/static
/templates/<service_id>
controller.py
service.py
```

This makes each implementation its own Flask `Blueprint`.

### `/static`
Because of Flask `Blueprint` imports, you can just add static files here and reference them (e.g. "static/img/blah.png") as if they were in the main `/static` files root dir.

### `/templates/<service_id>`
Again, Flask `Blueprint`s import the `/templates` directory as-is, but to avoid namespace collisions on the template files (e.g. `/templates/index.html`) they should be contained within a subdirectory named with the `Service.id` (e.g. `/templates/swan/index.html`)

### `Service` Implementation Class
Must inherit from `Service` and provide any additional functionality needed. The `Service` implementation class is meant to be the main hub for all things related to that particular `Service`. In general, external code would ideally only interact with the `Service` implementation class (e.g. )

### `controller.py`
Flask `Blueprint` for any endpoints required by this `Service`.

The coding philosophy should be to keep this code as simple as possible and keep most or all of the actual logic in the `Service` implementation class.

### Additional Files
The `SwanService` also includes an `api.py` to separate its back-end API calls from the user-facing `controller.py` endpoints. In general this is recommended to provide a clear separation.

An individual `Service` implementation may add whatever additional files or classes it needs.
