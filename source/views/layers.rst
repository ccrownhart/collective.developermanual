=========
Layers
=========

.. admonition:: Description
        
        Layers allow you to easily enable and disable views and other site functionality 
        based on installed add-ons and themes. 

.. contents :: :local:

Introduction
------------

Layers allow you to activate different code paths and modules depending on the external configuration

Examples:

* Theme code is only activated when theme is being selected

* Mobile browsing code is only activated when the site is being browser on a mobile phone

Layers are marker interfacse applied to HTTPRequest object. They are usually
used in conjunction with ZCML directives to dynamically activate various parts
of the configuration (theme files, add-on product functionality).

Layers ensure that only one add-on product can override the specific Plone instance functionality
in your site at a time, but still leaving you with an option to have possibly conflicting add-on products
in your buildout and ZCML once. Remember that multiple Plone site instances can share
the same ZCML and code files.

Many ZCML directives take the optional *layer* parameter. See example, resourceDirectory_

Layers can be activated when an add-on product is installed or a certain theme
is picked.

For more information, read

* `Making components theme specific <http://plone.org/documentation/manual/theme-reference/buildingblocks/components/themespecific>`_

* `Browser Layer tutorial <http://plone.org/documentation/tutorial/customization-for-developers/browser-layers>`_.

* `Zope 3 Developer Handbook, Skinning <http://zope3.xmu.me/skinning.html>`_

Using layers
------------

Some ZCML directives (example: `browser:page <http://apidoc.zope.org/++apidoc++/ZCML/http_co__sl__sl_namespaces.zope.org_sl_browser/page/index.html>`_) take layer attribute.

If you have

 # plonetheme.yourthemename.interfaces.IThemeSpecific layer defined in Python code

 # YourTheme product installed through add-on product installer on your site instance

Views and viewlets will be active on the site instance using the following ZCML::

     <!-- Site actions override in YourTheme -->
     <browser:viewlet
         name="plone.site_actions"
         manager="plone.app.layout.viewlets.interfaces.IPortalHeader"
         class=".siteactions.SiteActionsViewlet"
         layer="plonetheme.yourthemename.interfaces.IThemeSpecific"
         permission="zope2.View"
         />

Unconditional overrides
=======================

If you want to override a view or a viewlet unconditionally for all sites without the add-on product installer
support you need to use overrides.zcml.

Creating a layer
----------------

Theme layer
===========

Theme layers can be created with the following steps

1. Subclass an interface from IDefaultPloneLayer

   .. code-block:: python
    
       from plone.theme.interfaces import IDefaultPloneLayer
    
       class IThemeSpecific(IDefaultPloneLayer):
           """Marker interface that defines a Zope 3 skin layer bound to a Skin
              Selection in portal_skins.
              If you need to register a viewlet only for the "YourSkin"
              skin, this is the interface that must be used for the layer attribute
              in YourSkin/browser/configure.zcml.
           """

2. Register in in ZCML. Name must match the theme name.

   .. code-block:: xml
    
       <interface
           interface=".interfaces.IThemeSpecific"
           type="zope.publisher.interfaces.browser.IBrowserSkinType"
           name="SitsSkin"
           />

3. Declare your theme in profiles/default/skins.xml. Example.

   .. code-block:: xml
    
       <skin-path name="SitsSkin" based-on="Plone Default">
         <layer name="plone_skins_style_folder_name"
            insert-before="*"/>
       </skin-path>

4. Create profiles/default/browserlayer.xml.

   .. code-block:: xml
    
      <layers>
       <layer
           name="myproduct"
           interface="Products.myproduct.interfaces.IThemeSpecific"
           />
      </layers>

Add-on layer
=============

Add-on product layer is enabled when an add-on product is installed. 
Since one Zope application server may contain several Plone sites, 
you need separate different enanbled code paths by add-on layers -
otherwise all views and viewlets apply to all sites in one Zope application server. 

* You can enable views and viewlets specific to functional add-on

* Unlike theme layer, add-on layer depends on the activated add-on products, not on the selected theme

Add-on layer is a marker interface which is applied on :doc:`HTTP request object </serving/http_request_and_response>`
by Plone core logic.  

First create an an :doc:`interface </components/interfaces>` for your layer in ``your.product.interfaces.py``::

        """Define interfaces for your add-on.
        """
        
        import zope.interface
        
        class IAddOnInstalled(zope.interface.Interface):
            """A layer specific for this add-on product.
        
            This interface is referred in browserlayers.xml.
        
            All views and viewlets register against this layer will appear on your Plone site 
            only when the add-on installer has been run.
            """

You need to then refer to this in your add-on installer :doc:`setup profile </components/genericsetup>`.

``profile/default/browserlayer.xml`` file

.. code-block:: xml

        <layers>
         <layer
             name="your.product"
             interface="your.product.interfaces.IAddOnInstalled"
             />
        </layers>
                

.. note ::

        Add-on layer registry is persistent and stored in the database. The changes to add-on
        layers apply only when add-ons are installed or uninstalled.

More information

* http://pypi.python.org/pypi/plone.browserlayer

* See example in `LinguaPlone <https://github.com/plone/Products.LinguaPlone/tree/master/Products/LinguaPlone/profiles/default/browserlayer.xml>`_.

Using layers (for customization)
================================

Whole point of using layers is for someone else to override your ZCA registrations (for example a view).
By subclassing an marker interface for marker you can define more specific adapter which will take precedence
over primary adapter.


Manual layers
=============

Apply your layer to HTTPRequest in before_traverse hook or before you call
the code which looks up the interfaces.

Choosing skin layer dynamically 1: http://blog.fourdigits.nl/changing-your-plone-theme-skin-based-on-the-objects-portal_type

Choosing skin layer dynamically 2: http://code.google.com/p/plonegomobile/source/browse/trunk/gomobile/gomobile.mobile/gomobile/mobile/monkeypatch.py

See `plone.app.z3cform.z2 <http://svn.zope.org/plone.z3cform/trunk/plone/z3cform/z2.py?rev=88331&view=markup>`_ module.

In the example below we turn on a layer for request which is later checked by the rendering code.
This way some pages can ask special View/Viewlet rendering.

Example::

    # Defining layer

    from zope.publisher.interfaces.browser import IBrowserRequest

    class INoHeaderLayer(IBrowserRequest):
        """ When applied to HTTP request object, hedaer animations or images are not rendered on this.

        If this layer is on request do not render header images.
        This allows uncluttered editing of header animations and images.
        """

    # Applying layer for some requests (manually done in view)
    # The browser page which renders the form
    class EditHeaderAnimationsView(FormWrapper):

        form = HeaderCRUDForm

        def __call__(self):
            """ """

            # Signal viewlet layer that we are rendering
            # edit view for header animations and it is not meaningful
            # to try to render the big animation on this page
            zope.interface.alsoProvides(self.request, INoHeaderLayer)

            # Render the edit form
            return FormWrapper.__call__(self)
            
Problem with IDefaultBrowserLayer
==================================

``zope.publisher.interfaces.browser.IDefaultBrowserLayer`` is a problematic layer is it takes precedence in 
HTTP request multi-adapter look up (due to magic involving Plone themes).

Below is ``self.request.__provides__.__iro__`` dump for adding an extra form layer

.. code-block:: python

        (<InterfaceClass Products.CMFDefault.interfaces.ICMFDefaultSkin>, 
         <InterfaceClass plone.z3cform.z2.IFixedUpRequest>, 
         <InterfaceClass getpaid.expercash.browser.views.IExperCashFormLayer>,          
         <InterfaceClass plone.app.z3cform.interfaces.IPloneFormLayer>, 
         <InterfaceClass z3c.form.interfaces.IFormLayer>, 
         <InterfaceClass zope.publisher.interfaces.browser.IBrowserRequest>, 
         ...
         
One would assume a custom form layer (IExperCashFormLayer) is used and it would
take priority over more generic IPloneFormLayer. However, due to involment
of IDefaultBrowserLayer when registering items using ``<browser:page for="*">``
syntax.

The fix is to make your custom layer to subclass IDefaultBrowserLayer like::

        class IExperCashFormLayer(IDefaultBrowserLayer, IPloneFormLayer):
            """ Define a custom layer for which against our form macros are registered.
            
            This way we override the default plone.app.z3cform templates.
            
            Inheriting from IDefaultBrowserLayer makes sure this layer will get 1st priority.
            """ 

We register a custom macros like

.. code-block:: xml

          <!-- Override plone.app.z3cform default form template -->
          <browser:page
              name="ploneform-macros"
              for="*"
              layer=".views.IExperCashFormLayer"
              class=".views.Macros"
              template="templates/expercash-form-macros.pt"
              allowed_interface="zope.interface.common.mapping.IItemMapping"
              permission="zope.Public"
              />
             
            
And then the manual assignment works ok::

          def update(self):
                """ z3c.form.form.Form.Update() method
                """
                                
                # This will fix @@ploneform-macros to use our special version
                zope.interface.alsoProvides(self.request, IExperCashFormLayer)        
                
                # This should return macros we have registered
                macros = self.context.unrestrictedTraverse("@@ploneform-macros")            


(If this didn't make sense for you, don't worry. It doesn't make sense for me either.)            

Checking active layers
----------------------

Layers are activated on the current request object
================================================================

Example::

    if INoHeaderLayer.providedBy(self.request):
        # The page has asked to suspend rendering of the header animations
        return ""

Active themes and add-on products
======================================

registered_layers() method returns list of all layers active on the site.
Note that this is different list of layers which are application on the current
HTTP request object - request object may contain manually activated layers.

Example::

    from interfaces import IThemeSpecific


    from plone.browserlayer.utils import registered_layers

    if IThemeSpecific in registered_layers():
        # Your theme specific code
        pass
    else:
        # General code
        pass

Getting active theme layer
==========================

Only one theme layer can be activate at once.

Active theme name is defined in portal_skins properties.
This name can be resolved to a theme layer.


Debugging active layers
=======================

You can check the activated layers from HTTP request object in self.request.__provides__.__iro__.
Layers are evaluated from zero index (highest priority) the last index (lowest priority)

.. HTTPRequest: http://svn.zope.org/Zope/trunk/src/ZPublisher/HTTPRequest.py?rev=99866&view=markup

.. _resourceDirectory: http://apidoc.zope.org/++apidoc++/ZCML/http_co__sl__sl_namespaces.zope.org_sl_browser/resourceDirectory/index.html


Testing Layers
--------------

Plone testing toolkits won't register layers for you, you have to do it
yourself somehere in the bolierplate code::

	from zope.interface import directlyProvides

        directlyProvides(self.portal.REQUEST, IThemeLayer)
