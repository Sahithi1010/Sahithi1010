- router.post(
  '/users',
  check('name', 'Name is required').notEmpty(),
  check('email', 'Please include a valid email').isEmail(),
  check(
    'password',
    'Please enter a password with 6 or more characters'
  ).isLength({ min: 6 }),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    const { name, email, password , img } = req.body;

    try {
      let user = await User.findOne({ email });

      if (user) {
        return res
          .status(400)
          .json({ errors: [{ msg: 'User already exists' }] });
      }
    
      user = new User({
        name,
        email,
        avatar:img,
        password
      });
      
      const salt = await bcrypt.genSalt(10);

      user.password = await bcrypt.hash(password, salt);

      await user.save();

      const payload = {
        user: {
          id: user.id
        }
      };

      jwt.sign(
        payload,
        config.get('jwtSecret'),
        { expiresIn: '2 days' },
        (err, token) => {
          if (err) throw err;
          res.json({ token});
        }
      );
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server error');
    }
  }
);
router.get(
    '/user/:user_id',
    checkObjectId('user_id'),
    async ({ params: { user_id } }, res) => {

      try {
        let profile = await User.aggregate([
            {$match : {"_id" : ObjectId(user_id)}},
            {$lookup :{
                "from" : "posts",
                "localField" : "_id",
                "foreignField" : "user",
                "as"  : "posts"
            }},
            {$lookup:{
                "from" : "follows",
                "localField" : "_id",
                "foreignField" : "user",
                "as" : "friends"
            }},
            {$lookup:{
                "from" : "profiles",
                "localField" : "_id",
                "foreignField" : "user",
                "as" : "profile"
            }},
            {$project :{
                "id" :0,
                "password":0
            }}
        ])
        

        if (profile.length===0) return res.status(400).json({ msg: 'Profile not found' });
  
        return res.json(profile);
      } 
      catch (err) {
        console.error(err.message);
        return res.status(500).json({ msg: 'Server error' });
      }
    }
)
router.post(
  '/posts',
  auth,
  check('text', 'Text is required').notEmpty(),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }

    try {
      const user = await User.findById(req.user.id).select('-password');

      const newPost = new Post({
        text: req.body.text,
        name: user.name,
        avatar: user.avatar,
        user: req.user.id,
        img:req.body.img
      });

      const post = await newPost.save();

      res.json(post);
    } catch (err) {
      console.error(err.message);
      res.status(500).send('Server Error');
    }
  }
)
router.get('/posts/:id', auth, checkObjectId('id'), async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);

    if (!post) {
      return res.status(404).json({ msg: 'Post not found' });
    }

    res.json(post);
  } catch (err) {
    console.error(err.message);

    res.status(500).send('Server Error');
  }
})
