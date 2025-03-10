# Symfony OIDC bundle

This bundle can be used to add OIDC support to any Symfony application. We have only tested it with
SURFconext OIDC, but it should work with any OIDC provider!

Many thanks to https://github.com/jumbojett/OpenID-Connect-PHP for the implementation which this bundle uses although it has been modified to fix within an object oriented approach.

> Note that this repository is automatically mirrored from our own Gitlab instance.
> We will accept issues and merge requests here though!

### Version notes

Version 2 of this bundle only supports Symfony's new authentication manager, introduced in Symfony 5.3. As the security
manager matured in Symfony 5.4, that is the first version this bundle supports. Using the new authentication manager is
required for Symfony 6!

We also require the use of PHP8, as that significantly reduces the maintenance complexity.

Do you need this bundle, but you cannot enable the new authentication manager or use PHP8? Check out
the [v1.x](https://github.com/Drenso/symfony-oidc/tree/v1.x) branch and its documentation!

### Migrate from v1.x

Take a look at [UPGRADE.md](https://github.com/Drenso/symfony-oidc/blob/master/UPGRADE.md)!

### Installation

You can add this bundle by simply requiring it with composer:

```php
composer require drenso/symfony-oidc-bundle
```

If you're using Symfony Flex, your `.env` file should have been appended with some environment variables and
a `drenso_oidc.yaml` file should have been created in your configuration directory!

### Setup

##### OIDC Clients

Make sure to configure at least the default OIDC client in the `drenso_oidc.yaml` in your `config/packages` directory.
This can be done using the environment variables already added to your application by Symfony flex, or by updating the
configuration file. You can configure more clients, they will be available under the `drenso.oidc.client.{name}`, and are
autowirable by using `OidcClientInterface ${name}OidcClient`, for example `OidcClientInterface $defaultOidcClient`. If
the name does not match with one of the configured clients, the default client will be autowired.

Configuration example:

```yaml
drenso_oidc:
    #default_client: default # The default client, will be aliased to OidcClientInterface
    clients:
        default: # The client name, each client will be aliased to its name (for example, $defaultOidcClient)
            # Required OIDC client configuration
            well_known_url: '%env(OIDC_WELL_KNOWN_URL)%'
            client_id: '%env(OIDC_CLIENT_ID)%'
            client_secret: '%env(OIDC_CLIENT_SECRET)%'

            # Extra configuration options
            #redirect_route: '/login_check'
            #custom_client_headers: []

        # Add any extra client
        #link: # Will be accessible using $linkOidcClient
            #well_known_url: '%env(LINK_WELL_KNOWN_URL)%'
            #client_id: '%env(LINK_CLIENT_ID)%'
            #client_secret: '%env(LINK_CLIENT_SECRET)%'
```

##### User provider

You will need to update your User Provider to implement the methods from the `OidcUserProviderInterface`. Two methods
need to be implemented:

- `ensureUserExists(string $userIdentifier, OidcUserData $userData)`: Implement this method to bootstrap a new account
  using the data available from the passed `OidcUserData` object. The identifier is a configurable property from the
  user data, which defaults to `sub`. If the account cannot be bootstrapped, authentication will be impossible as the
  User Provider will not be capable of retrieving the user.
- `loadOidcUser(string $userIdentifier): UserInterface`: Implement this method to retrieve the user based on the
  identifier. We use a dedicated method instead of Symfony's default `loadUserByIdentifier` to allow you to detect where
  the login is coming from, without the need of creating a dedicated user provider. If the OIDC user identifiers are
  unique, a forward to the `loadUserByIdentifier` should be sufficient.

##### Firewall configuration

If you are using Symfony <6, make sure to enable the new authentication manager in the `security.yaml`:

```yaml
security:
  enable_authenticator_manager: true
```

Enable the `oidc` listener in the `security.yml` for your firewall:

```yaml
security:
  firewalls:
    main:
      pattern: ^/
      oidc: ~
```

There are a couple of options available for the `oidc` listener.

| Option                           | Default         | Description                                                                                                                                   |
|----------------------------------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| `check_path`                     | `/login_check`  | Only on this path the authenticator will accept authentication. Note that this should match with the redirect configured for the OIDC client. |
| `login_path`                     | `/login`        | The path to forward to when authentication is required                                                                                        | 
| `client`                         | `default`       | The configured OIDC client to use                                                                                                             |
| `user_identifier_property`       | `sub`           | The OidcUserData property to use as unique user identifier                                                                                    |
| `always_use_default_target_path` | `false`         | Used for the success handler                                                                                                                  |
| `default_target_path`            | `/`             | Used for the success handler                                                                                                                  |
| `target_path_parameter`          | `_target_path`  | Used for the success handler                                                                                                                  |
| `use_referer`                    | `false`         | Used for the success handler                                                                                                                  |
| `failure_path`                   | `null`          | Used for the failure handler                                                                                                                  |
| `failure_forward`                | `false`         | Used for the failure handler                                                                                                                  |
| `failure_path_parameter`         | `_failure_path` | Used for the failure handler                                                                                                                  |

You can configure them directly under the `oidc` listener in your firewall, for example the `user_identifier_property`:

```yaml
security:
  firewalls:
    main:
      oidc:
        user_identifier_property: email
```

##### Start the authentication

Use the controller example below to forward a user to the OIDC service:

```php
  /**
   * This controller forwards the user to the OIDC login
   *
   * @throws \Drenso\OidcBundle\Exception\OidcException
   */
  #[Route('/login_oidc', name: 'login_oidc')]
  #[IsGranted('PUBLIC_ACCESS')]
  public function surfconext(SessionInterface $session, OidcClientInterface $oidcClient): RedirectResponse
  {
    // Redirect to authorization @ OIDC provider
    return $oidcClient->generateAuthorizationRedirect();
  }
```

> It is possible to supply prompt and scope parameters to the `generateAuthorizationRedirect` method.

That should be all!
