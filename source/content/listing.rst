===================
 Listing objects
===================

.. admonition:: Description

        How to programmatically generate folder listings in Plone.

.. contents :: :local:

Introduction
------------

Plone has several methods of getting the list of folder items depending on whether

* You want to get all items or items visible for the currently logged in user

* You want to get hold of the item objects themselves or just indexed search information is enough
  (the latter is faster)
  
* You want to get Plone's contentish items only (contentItems) or Zope 2 management objects too (objectIds),
  the latter covers various site utilities found in the portal root and otherwise hidden magical items
    
Special attentation must be paid also on object ids. Zope looks all objects by traversing site graph
using ids. The id mapping is usually the property of a parent object, not child. Thus most of the listing 
methods tend to return (id, object) tuples instead of plain objects.   

Ensuring content item is a folder
---------------------------------

All Plone folderish content types provide ``IFolderish`` interface.
Check that this is present to make sure a content item is a 
folder and you can call ``contentItems()`` and other methods::


        from Products.CMFCore.interfaces import IFolderish
        
        def recurse_all_content(portal):
            
            output = StringIO()
            
            def recurse(context):
                """ Recurse through all content on Plone site """
                              
                print  >> output, "Recusring to item:" + str(context)
                
                # Make sure that we recurse to real folders only,
                # otherwise contentItems() might be acquired from higher level
                if IFolderish.providedBy(context):
                    for id, item in context.contentItems():
                        recurse(item)
                
            recurse(portal)
            
            return output
               

Getting all content objects inside a folder
-------------------------------------------

The method contentItems is defined in CMFCore/PortalFolder.py. From
Plone 4 onwards, you can also use folder.items() instead (this applies
to the whole section below).
See source code for details, e.g. filtering and other forms of listing

Querying folder through catalog
-------------------------------

These methods apply for real folders, and not for collections.

Getting indexed objects
========================

This is a faster method. You must have portal_catalog up-to-date for the folder.
This will return :doc:`brain objects </searching_and_indexing/query>`.

    brains = folder.getFolderContents()

Getting full objects
=====================

    items = folder.contentItems() # return Python list of children object tuples (id, object)

.. warning::

        contentItems() call may be costly, since it will return the actual content objects,
        not their indexed metadata from portal_catalog. You should avoid this method
        if possible.

.. warning::

        folder.contentItems() returns all items regardless of the user security context.

Getting folder objects filtered
-------------------------------

listFolderContents() method retrieves the full objects in the folder. It takes *contentFilter* as an argument which can be used to filter the results.  contentFilter uses the same syntax as portal_catalog queries but does not support all the same parameters e.g. "object_provides" is not supported. See the `ContentFilter class <http://svn.zope.org/Products.CMFCore/trunk/Products/CMFCore/PortalFolder.py?view=markup>`_ for details.

Example::

        # List all types in this folder whose portal_type is "CourseModulePage"

        return self.listFolderContents(contentFilter={"portal_type" : "CourseModulePage"})

.. warning::

	Security warning: listFolderContents() honors the currently logged in user roles.

.. warning::

	Performance warning: Slow for big folders. Preferably use portal_catalog and path based query
	to query items in a big folder.

Rules for filtering items
=========================

Plone applies some default rules for listFolderContents()

* portal_properties.nav_tree_properties.metaTypesNotToQuery: folders (large folders) don't generate listing

* :doc:`default_page </content/dynamic_views>` is not listed

* portal_properties.nav_tree_properties.: meta types marked here do not appear in the listing


Enforcing manual sort order
------------------------------

Below is an example how you sort content items by their manual sort order
(one you create via drag and drop on contents tab)::

        from OFS.interfaces import IOrderedContainer

        queried_objects = list(folder.listFolderContents())

        def get_position_in_parent(obj):
            """ 
            Use IOrderedContainer interface to extract the object's manual ordering position
            """
            parent = obj.aq_inner.aq_parent
            ordered = IOrderedContainer(parent, None)
            if ordered is not None:
                return ordered.getObjectPosition(obj.getId())
            return 0            

        def sort_by_position(a, b):
            """
            Python list sorter cmp() using position in parent.

            Descending order.
            """
            return get_position_in_parent(a) - get_position_in_parent(b)

        queried_objects = sorted(queried_objects, sort_by_position)


Getting object ids
-------------------

If you need to get ids only, use objectIds() method, or keys() in
Plone 4. This is a fast method::

    # Return a list of object ids in the folder
    ids = folder.objectIds()  # Plone 3 or older
    ids = folder.keys()       # Plone 4 or newer
    
    
.. warning::

        objectIds() and keys() will return ids for raw Zope 2 objects too, not just Plone content.
        If you run objectIds() in portal root, you will get objects like acl_users,
        portal_workflow and so on...

Getting non-contentish Zope objects
-------------------------------------

Manipulating non-contentish Zope objects 
are needed in some special cases. 

This listing method applies to all `OFS.Folder.Folder objects <http://svn.zope.org/Zope/trunk/src/OFS/interfaces.py?rev=96262&view=auto>`_, 
not just Plone content objects.

Example::

    for id, item in folder.objectItems():
        # id is 8-bit string of object id in the folder
        # item is the object itself
        pass

Checking for the existence of a particular object id
-----------------------------------------------------

If you want to know whether the folder has a certain item or not, you can use the following snippet.

Plone 4
========

Use has_key::

	if folder.has_key("my-object-id"):
		# Exists
	else:
		# Does not exist

Plone 3
========

There is a special case for Large Plone Folders (BTree based).
The following is optimal code, but you can simplify it if you don't need to check if the folder is BTreeFolder::

    # Use the BTreeFolder API if possible
    myid = "index_html"

    if base_hasattr(context, 'has_key'):
        # BTreeFolder's has_key returns numeric values
        return context.has_key(myid) and True or False
    elif myid in context.objectIds(): 
    # "elif myid in context:" in Plone 4 or newer
        return True
    else:
        return False
        
       
Listing the folder items using portal_catalog
---------------------------------------------

This should be your preferred method for querying folder items. portal_catalog searches are fast,
because they return catalog brain objects of real content objects (less database look ups).

.. warning::

        Returned catalog brain data, like Title, will be UTF-8 encoded.
        You need to call brain["title"].decode("utf-8") or similar
        to all strings you want to extract from the data.

Simple example how to get all items in a folder::

    # Get the physical path (includes Plone site name)
    # to the folder    
    path = folder.getPhysicalPath()
    
    # Convert getPhysicalPath() tuples result to 
    # slash separated string, which is used by ExtendedPathIndex
    path = "/".join(path)
    
    # This will fetch catalog brains.
    # Includes also unreleased items, not caring about workflow state.
    # depth = 1 means that subfolder items are not included
        
    brains = context.portal_catalog(path={"query" : path, "depth" : 1})


Complex example how to perform various filtering and honour some default
Plone filtering rules. This example is taken from Products.CMFPlone/skins/plone_scripts/getFolderContents::

    mtool = context.portal_membership
    cur_path = '/'.join(context.getPhysicalPath())
    path = {}

    if not contentFilter:
        # The form and other are what really matters
        contentFilter = dict(getattr(context.REQUEST, 'form',{}))
        contentFilter.update(dict(getattr(context.REQUEST, 'other',{})))
    else:
        contentFilter = dict(contentFilter)

    if not contentFilter.get('sort_on', None):
        contentFilter['sort_on'] = 'getObjPositionInParent'

    if contentFilter.get('path', None) is None:
        path['query'] = cur_path
        path['depth'] = 1
        contentFilter['path'] = path

    show_inactive = mtool.checkPermission('Access inactive portal content', context)

    # Evaluate in catalog context because some containers override queryCatalog
    # with their own unrelated method (Topics)
    contents = context.portal_catalog.queryCatalog(contentFilter, show_all=1,
                                                      show_inactive=show_inactive)

    if full_objects:
        contents = [b.getObject() for b in contents]

    if batch:
        from Products.CMFPlone import Batch
        b_start = context.REQUEST.get('b_start', 0)
        batch = Batch(contents, b_size, int(b_start), orphan=0)
        return batch

    return contents

Count of content items
-----------------------

Counting items using getFolderContents
======================================

The least expensive call for this for tens of items is to call len() for getFolderContents() which is portal_catalog based query::

        items = len(self.getFolderContents())

Counting items using contentItems
======================================

Alternative, if you know there are not many objects in in the folder,
you can call contentItems() (or simply items() in Plone 4 or newer) as this
will potentially wake less items than complex catalog query. 

.. warning:: 

        Security: This method does not consider access rights.  

Example (AT content class method)::

    def getMainImage(self):
        items = self.contentItems() # id, object tuples
        # "items = self.items()" in Plone 4 or newer
        if len(items) > 0:
            return items[1]
        

Custom folder listing
---------------------

Here is an example how to create a view which will render custom listing for a folder or a collection (ATTopic).

The view is called ProductSummaryView and it is registered with name *productsummary*. This example
is not suitable for your add-on product as is, but you need to tailor it for your specific needs.

.. warning::

    If you are going to call item/getObject for a catalog brain it might cause
    excessive database load as it causes a new database query per object.
    Try use information available in the catalog
    or add more catalog indexes. To know more about the
    issue read about waking up database objects.


* First let's register our view

    We could limit content types for which view is enabled by putting *Products.ATContentTypes.interface.IATFolder*
    or *Products.ATContentTypes.interface.IATTopic* into for attribute. The *configure.zcml* snippet below.

.. code-block:: xml

  <browser:page
      for="*"
      name="productcardsummary"
      class=".productcardsummaryview.ProductCardSummaryView"
      template="productcardsummaryview.pt"
      allowed_interface=".productcardsummaryview.IProductCardSummaryView"
      permission="zope2.View"
      />

* Below is the example view code, named as *productcardsummaryview.py*.

.. code-block:: python

    from zope.interface import implements, Interface

    from zope import schema

    from Products.Five import BrowserView
    from Products.CMFCore.utils import getToolByName

    from Products.ATContentTypes.interface import IATTopic

    # zope.18n message translator for your add-on product
    from yourproduct.namespace import appMessageFactory as _

    class IProductCardSummaryView(Interface):
        """ Allowed template variables exposed from the view.
        """

        # Item list as iterable Products.CMFPlone.PloneBatch.Batch object
        contents = schema.Object(Interface)


    class ProductCardSummaryView(BrowserView):
        """
        List summary information for all product cards in the folder.

        Batch results.
        """
        implements(IProductCardSummaryView)

        def query(self, start, limit, contentFilter):
            """ Make catalog query for the folder listing.

            @param start: First index to query

            @param limit: maximum number of items in the batch

            @param contentFilter: portal_catalog filtering dictionary with index -> value pairs.

            @return: Products.CMFPlone.PloneBatch.Batch object
            """

            # Batch size
            b_size = limit

            # Batch start index, zero based
            b_start = start

            # We use different query method, depending on
            # whether we do listing for topic or folder
            if IATTopic.providedBy(self.context):
                # ATTopic like content
                # Call Products.ATContentTypes.content.topic.ATTopic.queryCatalog() method
                # This method handles b_start internally and
                # grabs it from HTTPRequest object
                return self.context.queryCatalog(contentFilter, batch=True, b_size=b_size)
            else:
                # Folder or Large Folder like content
                # Call CMFPlone(/skins/plone_scripts/getFolderContents Python script
                # This method handles b_start parametr internally and grabs it from the request object
                return self.context.getFolderContents(contentFilter, batch=True, b_size=b_size)

        def __call__(self):
            """ Render the content item listing.
            """

            # How many items is one one page
            limit = 3

            # What kind of query we perform?
            # Here we limit results to ProductCard content type
            filter = { "portal_type" : "ProductCard" }

            # Read the first index of the selected batch parameter as HTTP GET request query parameter
            start = self.request.get("b_start", 0)

            # Perform portal_catalog query
            self.contents = self.query(start, limit, filter)

            # Return the rendered template (productcardsummaryview.pt), with content listing information filled in
            return self.index()

* Below is the corresponding page template skeleton productcardsummaryview.pt

.. code-block:: html

    <html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en"
          lang="en"
          metal:use-macro="here/main_template/macros/master"
          i18n:domain="yourproduct.namespace">
    <body>
        <div metal:fill-slot="main">
            <tal:main-macro metal:define-macro="main"
               tal:define="kssClassesView context/@@kss_field_decorator_view;
                           getKssClasses nocall:kssClassesView/getKssClassesInlineEditable;
                           ">


                <div tal:replace="structure provider:plone.abovecontenttitle" />

                <h1 metal:use-macro="here/kss_generic_macros/macros/generic_title_view">
                    Title or id
                </h1>

                <div tal:replace="structure provider:plone.belowcontenttitle" />

                <p metal:use-macro="here/kss_generic_macros/macros/generic_description_view">
                    Description
                </p>

                <div tal:replace="structure provider:plone.abovecontentbody" />

                <tal:listing define="batch view/contents">

                    <tal:block tal:repeat="item batch">
                        <div class="tileItem visualIEFloatFix vevent"
                             tal:define="item_url item/getURL|item/absolute_url;
                                               item_id item/getId|item/id;
                                               item_title_or_id item/pretty_title_or_id;
                                               item_description item/Description;
                                               item_type item/portal_type;
                                               item_type_title item/Type;
                                               item_type_class python: 'contenttype-' + normalizeString(item_type);
                                               item_modified item/ModificationDate;
                                               item_created item/CreationDate;
                                               item_wf_state        item/review_state|python: wtool.getInfoFor(item, 'review_state', '');
                                               item_wf_state_class python:'state-' + normalizeString(item_wf_state);
                                               item_creator item/Creator;
                                               item_start item/start/ISO|item/StartDate|nothing;
                                               item_end item/end/ISO|item/EndDate|nothing;
                                           "
                             tal:attributes="class string:tileItem visualIEFloatFix vevent ${item_type_class}">

                            <a href="#"
                               tal:attributes="href item_url">
                                <img src="" alt=""
                                     witdh="64"
                                     height="64"
                                     tal:condition="item_object/main_image|python:False"
                                     tal:attributes="src item_object/main_image" />
                            </a>


                            <h2 class="tileHeadline"
                                metal:define-macro="listitem">

                                <a href="#"
                                   class="summary url"
                                   tal:attributes="href item_url"
                                   tal:content="item_title_or_id">
                                    Item Title
                                </a>

                            </h2>

                            <p class="tileBody">
                                <span tal:omit-tag="" tal:condition="not:item_description">
                                    &nbsp;
                                </span>
                                <span class="description" tal:content="item_description">
                                    description
                                </span>
                            </p>

                            <p class="tileFooter">
                                <a href=""
                                   tal:attributes="href item_url"
                                   i18n:translate="read_more">
                                Read More&hellip;
                                </a>
                            </p>

                            <div class="visualClear"><!-- --></div>

                        </div>
                    </tal:block>

                    <!-- Navigation -->
                    <div metal:use-macro="here/batch_macros/macros/navigation" />

                </tal:listing>

                <div tal:replace="structure provider:plone.belowcontentbody" />

            </tal:main-macro>
        </div>
    </body>
    </html>

* Go to view page by adding /@@productsummary to your folder URL.

Making view to be available in Display... menu
==============================================

You need to add browser:menuItem entry to make your view appear in the Display... menu
where folders and topics can choose the style of the display.

See :doc:`dynamic views </content/dynamic_views>`.

You need to add

* <browser:menuItem> configuration directive with view id (e.g. @@productsummary)

* New properties to Folder.xml or Topic.xml so that view becomes available

Complex folder listings and filtering
--------------------------------------

The following example is for a very complex folder listing view.
You can call view methods to returns the listed items themselves and render 
the HTML in another view - this allows you to recycle this listing code easily.

The view does the various sanity checks what normal Plone item listings have

* No meta items

* No large folders

* No default views

* Filter by active language

* Do not list items where you do not have View permission

* The view also performs the listing on the parent container if the context itself
  is not folderish

Example code

.. code-block:: python

        class FolderListingView(BrowserView):
            """ Mobile folder listing helper view
        
            Use getItems() to get list of mobile folder listable items for automatically generated
            mobile folder listings (touch button list).
            """
        
            def getListingContainer(self):
                """ Get the item for which we perform the listing
                """
                context = self.context.aq_inner
                if IFolderish.providedBy(context):
                    return context
                else:
                    return context.aq_parent
        
            def getActiveTemplate(self):
                state = getMultiAdapter((self.context, self.request), name=u'plone_context_state')
                return state.view_template_id()
        
            def getTemplateIdsNoListing(self):
                """
                @return: List of mobile-specific ids found from portal_properties where not to show folder listing
                """
        
                try:
                    from gomobile.mobile.utilities import getCachedMobileProperties
                    context = aq_inner(self.context)
                    mobile_properties = getCachedMobileProperties(context, self.request)
                except:
                    mobile_properties = None
        
                return getattr(mobile_properties, "no_folder_listing_view_ids", [])
        
        
            def filterItems(self, container, items):
                """ Apply mobile specific filtering rules
        
                @param items: List of context brains
                """
        
                # Filter out default content
                default_page_helper = getMultiAdapter((container, self.request), name='default_page')
        
                portal_state = getMultiAdapter((container, self.request), name='plone_portal_state')
        
                # Active language
                language = portal_state.language()
        
                # Return  the default page id or None if not set
                default_page = default_page_helper.getDefaultPage(container)
                
                security_manager = getSecurityManager()
                
                meta_types_not_to_list = container.portal_properties.navtree_properties.metaTypesNotToList
                
        
                def show(item):
                    """ Filter whether the user can view a mobile item.
                    
                    @param item: Real content object (not brain)
        
                    @return: True if item should be visible in the listing
                    """
                
        
                    # Check from mobile behavior should we do the listing
                    try:
                        behavior = IMobileBehavior(item)
                        appearInFolderListing = behavior.appearInFolderListing
                    except TypeError:
                        # Site root or some weird object, give up
                        appearInFolderListing = True
        
                    if not appearInFolderListing:
                        # Default to appearing
                        return False
        
                    # Default page should not appear in the quick listing
                    if item.getId() == default_page:
                        return False
                    
                    if item.meta_type in meta_types_not_to_list:
                        return False
                    
                    # Two letter language code
                    item_lang = item.Language()
                    
                    # Empty string makes language netral content
                    if item_lang not in ["", None]:
                        if item_lang != language:
                            return False
        
                    # Note: getExcludeFromNav not necessarily exist on all content types 
                    if hasattr(item, "getExcludeFromNav"):                
                        if item.getExcludeFromNav():
                            return False
                        
                    # Does the user have a permission to view this object
                    if not security_manager.checkPermission(permissions.View, item):
                        return False
        
                    return True
        
                return [ i for i in items if show(i) == True ]
        
        
            def constructListing(self):
        
                # Iterable of content items for the item listing
                items = []
        
                # Check from mobile behavior should we do the listing
                try:
                    behavior = IMobileBehavior(self.context)
                    do_listing = behavior.mobileFolderListing
                except TypeError:
                    # Site root or some weird object, give up
                    do_listing = False
        
                # Do listing by default, must be explictly disabledc
                if not do_listing:
                    # No mobile behavior -> no mobile listing
                    return None
        
                container = self.getListingContainer()
        
                # Do not list if already doing folder listing
                template = self.getActiveTemplate()
                print "Active template id:" + template
                if template in self.getTemplateIdsNoListing():
                    # Listing forbidden by mobile rules
                    return None
        
        
                portal_properties = getToolByName(container, "portal_properties")
                navtree_properties = portal_properties.navtree_properties
                if container.meta_type in navtree_properties.parentMetaTypesNotToQuery:
                    # Big folder... listing forbidden
                    return None
                
                state = container.restrictedTraverse('@@plone_portal_state')
                        
                items = container.listFolderContents()
        
                items = self.filterItems(container, items)
        
                return items
        
            def getItems(self):
                """
                @return: Iterable of content objects. Never return None.
                """
                items = self.constructListing()
                if items == None:
                    return []
                return items

