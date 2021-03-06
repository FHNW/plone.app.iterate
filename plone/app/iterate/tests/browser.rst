Setup
-----

    >>> from plone.testing import z2
    >>> from plone.app.testing import login
    >>> from plone.app.testing import SITE_OWNER_NAME
    >>> from plone.app.testing import SITE_OWNER_PASSWORD

    >>> portal = layer['portal']
    >>> app = layer['app']
    >>> portal_url = portal.absolute_url()

    >>> browser = z2.Browser(app)
    >>> browser.handleErrors = False
    >>> browser.addHeader('Authorization',
    ...                   'Basic %s:%s' % (SITE_OWNER_NAME, SITE_OWNER_PASSWORD))

Create a document
-----------------

Go to our folder and create a document::

    >>> browser.open(portal.absolute_url())
    >>> browser.getLink('Add new').click()
    >>> 'Add new item' in browser.contents
    True
    >>> browser.getControl('Page').click()
    >>> browser.getControl('Add').click()
    >>> browser.getControl('Title').value = 'Hello, World!'
    >>> browser.getControl(name='text').value = 'Hello, World!'
    >>> browser.getControl('Save').click()
    >>> 'Changes saved.' in browser.contents
    True

Check it out
------------

Let's check out the document.  For this, we'll go to the *Check out*
form directly.  From there, we'll check out to the parent folder::

    >>> browser.getLink('Check out').click()
    >>> 'form.button.Checkout' in browser.contents
    True
    >>> browser.getControl(name='form.button.Checkout').click()
    >>> 'This is a working copy' in browser.contents
    True
    >>> browser.url
    'http://nohost/plone/copy_of_hello-world'

When viewing the original document, we should see a note that
someone's working on a working copy::

    >>> browser.open('http://nohost/plone/hello-world')
    >>> 'This item is being edited' in browser.contents
    True

We'll now edit the working copy and save it::

    >>> browser.open('http://nohost/plone/copy_of_hello-world')
    >>> browser.getLink('Edit').click()
    >>> browser.getControl('Title').value = 'Hello, World! version 2.0'
    >>> browser.getControl('Save').click()
    >>> 'Changes saved.' in browser.contents
    True
    >>> 'http://nohost/plone/copy_of_hello-world' in browser.url
    True

Check it in
-----------

Now that we've made our changes we'll check the working copy in::

    >>> browser.getLink('Check in').click()
    >>> browser.getControl('Check in').click()
    >>> 'Checked in' in browser.contents
    True
    >>> browser.url
    'http://nohost/plone/hello-world'
    >>> 'Hello, World! version 2.0' in browser.contents
    True

Permissions
-----------

For the community workflow, we expect Editors and Contributors to be able to
check out published pages::

    >>> portal.portal_workflow.setDefaultChain('plone_workflow')
    >>> ignore = portal.portal_workflow.updateRoleMappings()
    >>> browser.getLink("Publish").click()
    >>> "Item state changed" in browser.contents
    True

    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization', 'Basic editor:secret')

    >>> browser.open(portal.absolute_url() + '/hello-world')
    >>> browser.getLink("Check out").click()
    >>> 'form.button.Checkout' in browser.contents
    True
    >>> browser.getControl(name='form.button.Checkout').click()
    >>> "Check-out created" in browser.contents
    True

They should not, however, be able to check in their working copy
again.  That's because the item is in the ``published`` state and
therefore our Editor lacks permissions to modify the original::

    >>> browser.getLink("Cancel check-out")
    <Link ...>
    >>> browser.getLink("Check in")
    Traceback (most recent call last):
    ...
    LinkNotFoundError

The Editor could, however, ask someone to retract the original so he
gains permissions again and check in (and then possibly request for
review)::

    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization',
    ...                   'Basic %s:%s' % (SITE_OWNER_NAME, SITE_OWNER_PASSWORD))
    >>> browser.open(portal.absolute_url() + '/hello-world')
    >>> browser.getLink("Published").click()
    >>> browser.getControl("Retract").click()
    >>> browser.getControl("Save").click()
    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization', 'Basic editor:secret')
    >>> browser.open(portal.absolute_url() + '/hello-world')
    >>> browser.getLink("working copy").click()
    >>> browser.getLink("Check in").click()
    >>> browser.getControl("Check in").click()
    >>> "Checked in" in browser.contents
    True

Folders
-------

Turn on versioning for folders::

    >>> from Products.CMFCore.utils import getToolByName
    >>> tool = getToolByName(portal, 'portal_repository')
    >>> tool.addPolicyForContentType('Folder', u'at_edit_autoversion')
    >>> tool.addPolicyForContentType('Folder', u'version_on_revert')
    >>> versionable_types = tool.getVersionableContentTypes()
    >>> versionable_types.append('Folder')
    >>> tool.setVersionableContentTypes(versionable_types)

Go to our folder and create a folder::

    >>> browser = z2.Browser(app)
    >>> browser.handleErrors = False
    >>> browser.addHeader('Authorization',
    ...                   'Basic %s:%s' % (SITE_OWNER_NAME, SITE_OWNER_PASSWORD))
    >>> browser.open(portal.absolute_url())
    >>> browser.getLink('Folder').click()
    >>> browser.getControl('Title').value = 'Foo Folder'
    >>> browser.getControl('Save').click()
    >>> 'Changes saved.' in browser.contents
    True

Add an item to the folder::

    >>> browser.getLink('Foo Folder').click()
    >>> browser.getLink('Page').click()
    >>> browser.getControl('Title').value = 'Bar Page'
    >>> browser.getControl('Save').click()
    >>> 'Changes saved.' in browser.contents
    True

Check out the folder::

    >>> browser.getLink('Foo Folder').click()
    >>> browser.getLink('Check out').click()
    >>> 'form.button.Checkout' in browser.contents
    True
    >>> browser.getControl(name='form.button.Checkout').click()
    >>> 'This is a working copy' in browser.contents
    True
    >>> wc_url = browser.url
    >>> wc_url
    'http://nohost/plone/copy_of_foo-folder'

Add another item to the checked out copy::

    >>> browser.getLink(url='Document').click()
    >>> browser.getControl('Title').value = 'Qux Page'
    >>> browser.getControl('Save').click()
    >>> 'Changes saved.' in browser.contents
    True

Now that we've added another item, check the working copy in::

    >>> browser.open(wc_url)
    >>> browser.open(browser.url + '/@@content-checkin')
    >>> browser.getControl('Check in').click()
    >>> 'Checked in' in browser.contents
    True
    >>> browser.url
    'http://nohost/plone/foo-folder'
    >>> browser.getLink('Qux Page')
    <Link text='Qux Page' url='http://nohost/plone/foo-folder/qux-page'>

Bugs
----

The "Cancel check-out" action should not be present on items that are
not checked out (#8735)::

    >>> browser.getLink("Cancel check-out")
    Traceback (most recent call last):
    ...
    LinkNotFoundError

Some items, like the Plone site root, don't do references.  This broke
the condition for the "Cancel check-out" action on these items
(#8737)::

    >>> z2.login(layer['app']['acl_users'], SITE_OWNER_NAME)
    >>> if 'front-page' in portal:
    ...     portal.manage_delObjects(['front-page'])
    >>> browser.open(portal.absolute_url())

Working copy workflows
----------------------

It's possible to assign a different workflow to working copies in combination
with Products.CMFPlacefulWorkflow.  This usually makes sense: you should be
checking in a working copy rather than publishing it.

We have a working copy workflow defined in our textfixture profile.  To enable
you need to set a couple of registry-entries::

    >>> browser.addHeader('Authorization',
    ...                   'Basic %s:%s' % (SITE_OWNER_NAME, SITE_OWNER_PASSWORD))
    >>> browser.open("http://nohost/plone/portal_registry/edit/plone.app.iterate.interfaces.IIterateSettings.checkout_workflow_policy")
    >>> browser.getControl(name="form.widgets.value").value
    'checkout_workflow_policy'
    >>> browser.getControl(name="form.widgets.value").value = 'working-copy'
    >>> browser.getControl(name="form.buttons.save").click()
    >>> browser.open("http://nohost/plone/portal_registry/edit/plone.app.iterate.interfaces.IIterateSettings.enable_checkout_workflow")
    >>> browser.getControl(name="form.widgets.value:list").value
    []
    >>> browser.getControl(name="form.widgets.value:list").controls[0].selected = True
    >>> browser.getControl(name="form.buttons.save").click()

Create a new page to test workflows with::

    >>> browser.open(portal.absolute_url())
    >>> browser.getLink('Add new').click()
    >>> 'Add new item' in browser.contents
    True
    >>> browser.getControl('Page').click()
    >>> browser.getControl('Add').click()
    >>> browser.getControl('Title').value = 'My workflow test'
    >>> browser.getControl(name='text').value = 'My workflow test'
    >>> browser.getControl('Save').click()
    >>> 'Changes saved.' in browser.contents
    True
    >>> workflow_test_url = browser.url

Checkout::

    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization', 'Basic contributor:secret')
    >>> browser.open(workflow_test_url)
    >>> browser.getLink(id='plone-contentmenu-actions-iterate_checkout').click()
    >>> browser.contents
    '...Check out...My workflow test...'
    >>> checkout_form = browser.getForm(name='checkout')
    >>> checkout_form.getControl('Parent folder').selected = True
    >>> checkout_form.getControl('Check out').click()
    >>> browser.contents
    '...This is a working copy of...My workflow test..., made by...contributor...'
    >>> browser.contents
    '...state-draft-copy...'
    >>> workflow_checkout_url = browser.url

Check get info message on original::

    >>> browser.open(workflow_test_url)
    >>> browser.contents
    '...This item is being edited by...contributor...a working copy...'

We're going to manually give the contributor user the CheckoutPermission
to check it's used when displaying the info messages.  In our workflow
once the checked out item is submitted the contributor no longer has
permission to modify it but we still want them to see the info messages::

    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization',
    ...                   'Basic %s:%s' % (SITE_OWNER_NAME, SITE_OWNER_PASSWORD))

    >>> from plone.app.iterate.permissions import CheckoutPermission
    >>> browser.open('{0}/manage_permissionForm?permission_to_manage={1}'.format(portal.absolute_url(), CheckoutPermission))
    >>> browser.getControl(name='roles:list').value = browser.getControl(name='roles:list').value + ['Contributor']
    >>> browser.getControl('Save Changes').click()

    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization', 'Basic contributor:secret')
    >>> browser.open(workflow_checkout_url)
    >>> browser.getLink(id='workflow-transition-submit-copy-for-publication')\
    ...     .click()
    >>> browser.contents
    '...state-pending-copy...'
    >>> browser.contents
    '...This is a working copy of...My workflow test..., made by...contributor...'
    >>> browser.open(workflow_test_url)
    >>> browser.contents
    '...This item is being edited by...contributor...a working copy...'

Check security permisions on workflow have been applied.  We remove copy or
move permissions in our workflow so this should not appear in the action menu.
http://code.google.com/p/dexterity/issues/detail?id=258 ::

    >>> browser = z2.Browser(app)
    >>> browser.addHeader('Authorization', 'Basic editor:secret')
    >>> browser.open(workflow_checkout_url)
    >>> browser.getLink(id='plone-contentmenu-actions-copy')
    Traceback (most recent call last):
    ...
    LinkNotFoundError
    >>> browser.getLink(id='plone-contentmenu-actions-delete')
    <Link text='Delete' ...>
