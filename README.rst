=========
tesserocr
=========

A simple, |Pillow|_-friendly,
wrapper around the ``tesseract-ocr`` API for Optical Character Recognition
(OCR).

**tesserocr** integrates directly with Tesseract's C++ API using Cython
which allows for a simple Pythonic and easy-to-read source code. It
enables real concurrent execution when used with Python's ``threading``
module by releasing the GIL while processing an image in tesseract.

**tesserocr** is designed to be |Pillow|_-friendly but can also be used
with image files instead.

.. |Pillow| replace:: ``Pillow``
.. _Pillow: http://python-pillow.github.io/

Requirements
============

Requires libtesseract (>=3.04) and libleptonica.

On Debian/Ubuntu:

::

    $ apt-get install tesseract-ocr libtesseract-dev libleptonica-dev

Optionally requires ``Cython`` for building (otherwise the generated
.cpp file is compiled) and |Pillow|_ to support ``PIL.Image`` objects.

Installation
============

::

    $ pip install tesserocr

Usage
=====

Initialize and re-use the tesseract API instance to score multiple
images:

.. code:: python

    from tesserocr import PyTessBaseAPI

    images = ['sample.jpg', 'sample2.jpg', 'sample3.jpg']

    with PyTessBaseAPI() as api:
        for img in images:
            api.SetImageFile(img)
            print api.GetUTF8Text()
            print api.AllWordConfidences()
    # api is automatically finalized when used in a with-statement (context manager).
    # otherwise api.End() should be explicitly called when it's no longer needed.

``PyTessBaseAPI`` exposes several tesseract API methods. Make sure you
read their docstrings for more info.

Basic example using available helper functions:

.. code:: python

    import tesserocr
    from PIL import Image

    print tesserocr.tesseract_version()  # print tesseract-ocr version
    print tesserocr.get_languages()  # prints tessdata path and list of available languages

    image = Image.open('sample.jpg')
    print tesserocr.image_to_text(image)  # print ocr text from image
    # or
    print tesserocr.file_to_text('sample.jpg')

``image_to_text`` and ``file_to_text`` can be used with ``threading`` to
concurrently process multiple images which is highly efficient.

Advanced API Examples
---------------------

GetComponentImages example:

.. code:: python

    from PIL import Image
    from tesserocr import PyTessBaseAPI

    image = Image.open('/usr/src/tesseract/testing/phototest.tif')
    with PyTessBaseAPI() as api:
        api.SetImage(image)
        boxes = api.GetComponentImages(RIL.TEXTLINE, True)
        print 'Found {} textline image components.'.format(len(boxes))
        for i, (im, box, _, _) in enumerate(boxes):
            # im is a PIL image object
            # box is a dict with x, y, w and h keys
            api.SetRectangle(box['x'], box['y'], box['w'], box['h'])
            ocrResult = api.GetUTF8Text()
            conf = api.MeanTextConf()
            print (u"Box[{0}]: x={x}, y={y}, w={w}, h={h}, "
                   "confidence: {1}, text: {2}").format(i, conf, ocrResult, **box)

Orientation and script detection (OSD):

.. code:: python

    from PIL import Image
    from tesserocr import PyTessBaseAPI, PSM

    with PyTessBaseAPI(psm=PSM.AUTO_OSD) as api:
        image = Image.open("/usr/src/tesseract/testing/eurotext.tif")
        api.SetImage(image)
        api.Recognize()

        it = api.AnalyseLayout()
        orientation, direction, order, deskew_angle = it.Orientation()
        print "Orientation: {:d}".format(orientation)
        print "WritingDirection: {:d}".format(direction)
        print "TextlineOrder: {:d}".format(order)
        print "Deskew angle: {:.4f}".format(deskew_angle)

Iterator over the classifier choices for a single symbol:

.. code:: python

    from tesserocr import PyTessBaseAPI, RIL, iterate_level

    with PyTessBaseAPI() as api:
        api.SetImageFile('/usr/src/tesseract/testing/phototest.tif')
        api.SetVariable("save_blob_choices", "T")
        api.SetRectangle(37, 228, 548, 31)
        api.Recognize()

        ri = api.GetIterator()
        level = RIL.SYMBOL
        for r in iterate_level(ri, level):
            symbol = r.GetUTF8Text(level)  # r == ri
            conf = r.Confidence(level)
            if symbol:
                print u'symbol {}, conf: {}'.format(symbol, conf),
            indent = False
            ci = r.GetChoiceIterator()
            for c in ci:
                if indent:
                    print '\t\t ',
                print '\t- ',
                choice = c.GetUTF8Text()  # c == ci
                print u'{} conf: {}'.format(choice, c.Confidence())
                indent = True
            print '---------------------------------------------'