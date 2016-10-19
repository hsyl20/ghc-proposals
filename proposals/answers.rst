@mboes

My real world example:
Current ``Storable`` class is defined as::

   class Storable a where
      sizeOf :: a -> Int
      peek   :: Ptr a -> IO a
      ...

We can automatically generate instances from a ``Generic`` instance (cf c-storable-deriving package) with default signatures::

   class Storable a where
      sizeOf :: a -> Int
      peek   :: Ptr a -> IO a

      default sizeOf :: (Generic a, GStorable (Rep a)) => a -> Int
      sizeOf = genericSizeOf

      default peek :: (Generic a, GStorable (Rep a)) => Ptr a -> IO a
      peek = genericPeek
      ...

Issue 1) I want to replace ``sizeOf`` with a type-level literal (to compute offsets at compile time, among other things)::

   class Storable a where
      type SizeOf a :: Nat
      peek   :: Ptr a -> IO a

AFAIK, we don't have the equivalent of default signatures for associated type families, so I can't define::

   class Storable a where
      type SizeOf a :: Nat
      default type (Generic a, GStorable (Rep a)) => SizeOf a = ComputeSizeOf a

Issue 2) I want to support several ways to store a datat type, namely ``Struct`` and ``PackedStruct``, with ``Struct`` being the default. A solution would be to allow several default signatures, but it isn't supported either (see #7395)::

   class Storable a where
      peek :: Ptr a -> IO a

      default peek :: ( Generic a, GStorable (Rep a)
                      , Layout a ~ Struct) => Ptr a -> IO a
      peek = genericPeekStruct

      default peek :: ( Generic a, GPackedStorable (Rep a)
                      , Layout a ~ PackedStruct) => Ptr a -> IO a
      peek = genericPeekPackedStruct

      default peek :: (Generic a, GStorable (Rep a)) => Ptr a -> IO a
      peek = genericPeekStruct

It seems to me that default signatures already are a kind of instance
selection based on the context. But it is hard to compose with them (e.g.,
add default signatures to a class without modifying it, overload them, mix
them with default methods, etc.). So with the proposal, I wouldn't use
them but instead I could write a ``peek' :: Ptr a -> IO a`` method that
would test in sequence:

1) If (Fulfilled (Storable a)), then use Storable's peek method
2) If (Fulfilled (Generic a, HasStorageMethod a)), then use ``genericPeek @(StorageMethod a)``
3) If (Fulfilled (Generic a)), then use ``genericPeek @Struct``
4) TypeError

Where::
   class HasStorageMethod a where
      type StorageMethod a :: *


