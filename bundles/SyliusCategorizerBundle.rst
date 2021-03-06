SyliusCategorizerBundle
=======================

Categorizing whatever you want just got easier. Grouping products, posts or any other model is common feature in most of modern web applications.
So why implement it every time you need it? You can use this bundle to create multiple categorized catalogs of any object.
It provides all controllers, routing, base mapping and services that will boost you development.

Features
--------

* Base support for many different persistence layers. Currently only Doctrine ORM driver is implemented.
* Allows you to create custom ordered flat list of categories, default controllers and forms will handle CRUD and moving up/down the categories.
* Thanks to `Doctrine Extensions library <http://github.com/l3pp4rd/DoctrineExtensions>`_ you can have nested set of categories, just extend proper class, modify form, add little mapping and it works.
* Handles both many-to-one and many-to-many relations between objects and the categories. Bundle will check it for you.
* You can create as many catalogs as you want, by `catalog` we understand set of categories and the items, for example products or blog posts.
* It uses `Pagerfanta <https://github.com/whiteoctober/Pagerfanta>`_ to paginate over the category items, but you can easily disable the pagination for specific catalog.
* Thanks to awesome `Symfony2 <http://symfony.com>`_ everything is configurable and extensible.
* Unit tested.

Installation
------------

Recommended tool for managing dependencies for Sylius bundles is `Composer <http://getcomposer.org>`_.

Installation via Composer
~~~~~~~~~~~~~~~~~~~~~~~~~

If you don't know what is Composer, read `this guide <http://getcomposer.org/doc/00-intro.md>`_.

Create a `composer.json` file in your project root and add this.

.. code-block:: json

    {
        "require": {
            "sylius/categorizer-bundle": "*"
        }
    }

Then, download composer and install deps.

.. code-block:: bash

    $ wget http://getcomposer.org/composer.phar
    $ php composer.phar install

This should download all required libraries and the bundle itself.
You can use the Composer autoloader or define paths manually in your own `autoload.php`.

Downloading the bundle manually
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The good practice is to download it to `vendor/bundles/Sylius/Bundle/CategorizerBundle`.

This can be done in several ways, depending on your preference.

Using the vendors script
************************

If you still rely on old vendors script from 2.0 version of Symfony, follow this steps.
Add the following lines in your `deps` file. ::

    [SyliusCategorizerBundle]
        git=git://github.com/Sylius/SyliusCategorizerBundle.git
        target=bundles/Sylius/Bundle/CategorizerBundle

Now, run the vendors script to download the bundle.

.. code-block:: bash

    $ php bin/vendors install

Using submodules
****************

If you prefer instead to use git submodules, then run the following lines.

.. code-block:: bash

    $ git submodule add git://github.com/Sylius/SyliusCategorizerBundle.git vendor/bundles/Sylius/Bundle/CategorizerBundle
    $ git submodule update --init

Autoloader configuration
************************

.. note::

    If you use the autoloader generated by Composer, you obviously can skip this step.

Add the `Sylius\\Bundle` namespace to your autoloader.

.. code-block:: php

    <?php

    // app/autoload.php

    $loader->registerNamespaces(array(
        'Sylius\\Bundle' => __DIR__.'/../vendor/bundles'
    ));

Adding bundle to kernel
~~~~~~~~~~~~~~~~~~~~~~~

Finally, enable the bundle in the kernel.

.. code-block:: php

    <?php

    // app/AppKernel.php
    public function registerBundles()
    {
        $bundles = array(
            // ...
            new Sylius\Bundle\CategorizerBundle\SyliusCategorizerBundle(),
        );
    }

Importing routing configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now is the time to import routing files. Open up your `routing.yml` file and add those lines.
Customize the prefixes or whatever you want.

.. code-block:: yaml

    sylius_categorizer_category:
        resource: @SyliusCategorizerBundle/Resources/config/routing/frontend/category.yml

    sylius_categorizer_backend_category:
        resource: @SyliusCategorizerBundle/Resources/config/routing/backend/category.yml
        prefix: /administration

Usage guide
-----------

.. note::

    The bundle requires at least one catalog created.

`Sylius sandbox application <http://github.com/Sylius/Sylius-Sandbox>`_ is a great example of this bundle usage.

There are two configured catalogs, one simple categories set for blog posts and one nested set of product categories.
You can try it by installing the sandbox or check the sources, but here we'll implement both catalogs from scratch.

Many to one relation between categories and items
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Imagine that you have your Product model and you want to have them grouped in categories.
First, you need to decide what relation to use, in this part we will will allow to store each product only in one category.
No worries, in next part you'll see how to do this with many-to-many. Now, let's create our simple category.

It's recommended to put it in the same namespace as the product entity, as you won't need any additional bundle or services except SyliusCategorizerBundle.

.. code-block:: php

    <?php

    namespace Acme\Bundle\AssortmentBundle\Entity;

    use Sylius\Bundle\CategorizerBundle\Entity\NestedCategory;

    class Category extends BaseCategory
    {
        private $products; // remember the name of this property!

        public function getProducts()
        {
            return $this->products;
        }

        public function setProducts(Collection $products)
        {
            $this->products = $products;
        }

        // you can of course implement other methods like `addProduct` but it's not important for our example.
    }

Now we need to add two simple methods to product entity.

.. code-block:: php

    <?php

    namespace Acme\Bundle\AssortmentBundle\Entity;

    use Sylius\Bundle\CategorizerBundle\Model\CategoryInterface;

    class Product
    {
        // your properties.

        private $category;

        public function getCategory()
        {
            return $this->category;
        }

        public function setCategory(CategoryInterface $category)
        {
            $this->category = $category;
        }
    }

That's nothing special, just a simple relation, with which you're probably familiar.
Now we need to add some mapping for both of our classes.
Let's start with Category. You can of course use any other mapping driver, but for our example we'll use XML.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8" ?>

    <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
                      xmlns:gedmo="http://gediminasm.org/schemas/orm/doctrine-extensions-mapping"
    >

        <entity name="Acme\Bundle\AssortmentBundle\Entity\Category"
                table="acme_assortment_category"
                repository-class="Gedmo\Tree\Entity\Repository\NestedTreeRepository">
            <order-by>
                <order-by-field name="treeLeft" direction="DESC" />
            </order-by>
            <id name="id" column="id" type="integer">
                <generator strategy="AUTO" />
            </id>
            <field name="treeLeft" column="tree_left" type="integer">
                <gedmo:tree-left />
            </field>
            <field name="treeRight" column="tree_right" type="integer">
                <gedmo:tree-right />
            </field>
            <field name="treeLevel" column="tree_level" type="integer">
                <gedmo:tree-level />
            </field>
            <one-to-many field="products" target-entity="Acme\Bundle\AssortmentBundle\Entity\Product" mapped-by="category" />
            <one-to-many field="children" target-entity="Acme\Bundle\AssortmentBundle\Entity\Category" mapped-by="parent">
            <order-by>
                <order-by-field name="treeLeft" direction="ASC" />
            </order-by>
            </one-to-many>
            <many-to-one field="parent" target-entity="Acme\Bundle\AssortmentBundle\Entity\Category">
                <join-column name="parent_id" referenced-column-name="id" on-delete="SET NULL"/>
                <gedmo:tree-parent />
            </many-to-one>
            <gedmo:tree type="nested" />
        </entity>

    </doctrine-mapping>

Don't be scared by all those mappings, they're required for having nested set of categories.

The most important for you is the products mapping.
Next step is adding much simpler mapping for product, to fully tie products with categories.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8" ?>

    <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
                      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                      xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                                          http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd"
    >

        <entity name="Acme\Bundle\AssortmentBundle\Entity\Product" table="acme_assortment_product">
            <id name="id" column="id" type="integer">
                <generator strategy="AUTO" />
            </id>
            <many-to-one field="category" target-entity="Acme\Bundle\AssortmentBundle\Entity\Category">
                <join-column name="category_id" referenced-column-name="id" />
            </many-to-one>
        </entity>

    </doctrine-mapping>

Enough messing with XML, now we have our classes and mappings set, let's configure our first catalog!
Open up your **config.yml** inside *app/config* directory and add this cofiguration for the bundle.
Pay attention to comments, they will answer a lot of your questions.

.. code-block:: yml

    sylius_categorizer:
        driver: doctrine/orm
        catalogs:
            assortment:
                property: "products"
                model: Acme\Bundle\AssortmentBundle\Entity\Category
                form: acme_assortment_category
                templates:
                    backend:
                        list: AcmeAssortmentBundle:Backend/Category:list.html.twig
                        show: AcmeAssortmentBundle:Backend/Category:show.html.twig
                        create: AcmeAssortmentBundle:Backend/Category:create.html.twig
                        update: AcmeAssortmentBundle:Backend/Category:update.html.twig
                    frontend:
                        list: AcmeAssortmentBundle:Frontend/Category:list.html.twig
                        show: AcmeAssortmentBundle:Frontend/Category:show.html.twig

.. code-block:: php

    <?php

    namespace Acme\Bundle\AssortmentBundle\Form\Type;

    use Sylius\Bundle\CategorizerBundle\Form\Type\CategoryType as BaseCategoryType;
    use Symfony\Component\Form\FormBuilder;

    class CategoryType extends BaseCategoryType
    {
        public function buildForm(FormBuilder $builder, array $options)
        {
            parent::buildForm($builder, $options);

            $builder
                ->add('parent', 'sylius_categorizer_category_choice', array(
                    'required'      => false,
                    'multiple'      => false,
                    'catalog'       => 'assortment'
                ))
            ;
        }

        public function getName()
        {
            return 'acme_assortment_category';
        }
    }

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8" ?>

    <container xmlns="http://symfony.com/schema/dic/services"
               xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:schemaLocation="http://symfony.com/schema/dic/services
                                   http://symfony.com/schema/dic/services/services-1.0.xsd"
    >

        <services>
            <service id="acme_assortment.form.type.category" class="Acme\Bundle\AssortmentBundle\Form\Type\CategoryType">
                <tag name="form.type" alias="acme_assortment_category" />
            </service>
        </services>

    </container>
