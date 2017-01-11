#基于Django和React.js开发的一个投票系统（2）——API设计与实现
===
##设计

回顾一下流程图，不难整理出前端与后台需要交互的数据

- 身份验证
 + 发送：学号和信息门户密码
 + 接收：登录成功 或 密码错误 或 今日已投过票
- 获取作品列表
 + 发送：组别
 + 接收：该组别作品列表
- 投票
 + 发送：选择的作品
 + 接收：投票成功 或 投票失败
 
##实现

###身份验证及登录

    def get_token(request):
        """
        Try to get a token and login
        """
        school_id = request.POST['sid']
        password = request.POST['password']
    
        # check whether the activity has started
        if datetime.datetime.now() > datetime.datetime(2016, 11, 16, 0):
            return JsonResponse({'info': 0})
    
        # check whether the user has voted
        if is_vote_today(school_id):
            return JsonResponse({'info': 1})
    
        # check whether the student who use this student number has been generated the User model
        user_set = User.objects.filter(username=school_id)
        if user_set:
            user = user_set[0]
        else:
            if whu_student_check(school_id, password):
                # authentication succeeded and generate User model
                user = generate_user(school_id, password)
            else:
                # authentication failed and return login failed
                return JsonResponse({'info': 0})
    
        # check whether there is a valid token, or create new one
        token_set = Token.objects.filter(user_id=user.id)
        if token_set:
            token = token_set[0].key
        else:
            token = create_token(user)
        return JsonResponse({'token': token})
    
    
    def create_token(user):
        """
        create new token for user
        """
        token = Token.objects.create(user=user)
        print(token.key)
        return token.key
    
    
    def generate_user(school_id, password):
        """
        generate new user model
        """
        user = User.objects.create_user(
            school_id,
            '',
            password
        )
        return user
    
    
    def whu_student_check(school_id, password):
        """
        check whether the user is a Wuhan university student
        """
        url = "http://www.whusu.com.cn/check.php?sid=" + school_id + "&password=" + password
        r = requests.get(url)
        if r.text == '1':
            return 1
        else:
            return 0
    
    
    def is_vote_today(sid):
        """
        check whether the user has voted today
        """
        vote_set = VoteItem.objects.filter(school_id=sid)
        for item in vote_set:
            item_create_time = item.create_time + datetime.timedelta(hours=8)
            item_date = item_create_time.date()
            if item_date == datetime.datetime.now().date():
                return True
        return False
        

##获取作品列表

    # serializers
    class PhotographicWorkItemSerializer(serializers.ModelSerializer):
        class Meta:
            model = PhotographicWorkItem
            fields = ('id', 'name', 'group', 'vote', 'photos', 'sort')
    
    
    class PhotographicWorkItemViewSet(viewsets.ModelViewSet):
        """
        get photo graphic work item list
        """
        queryset = PhotographicWorkItem.objects.all().order_by('sort')
        serializer_class = PhotographicWorkItemSerializer
        permission_classes = [AllowAny, ]
        filter_backends = (filters.DjangoFilterBackend,)
        filter_fields = ('group', 'name')

##投票

    class VoteItemCreate(APIView):
        """
        create vote item
        """
        authentication_classes = (TokenAuthentication,)
        permission_classes = [IsAuthenticatedOrReadOnly, ]
    
        # def get(self, request, format=None):
        #     vote_item = VoteItem.objects.all()
        #     serializer = VoteItemSerializer(vote_item, many=True)
        #     return Response(serializer.data)
    
        def post(self, request, format=None):
            vote_num_today = count_today_vote(request.data['school_id'])
    
            if vote_num_today >= 10:
                return JsonResponse({'info': 1})
            if is_same_vote_item_today(request.data['school_id'], request.data['photographic_work_item']):
                return JsonResponse({'info': 2})
    
            serializer = VoteItemSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    
    def count_today_vote(sid):
        """
        get the number of votes cast today
        """
        num = 0
        vote_set_for_user = VoteItem.objects.filter(school_id=sid)
        for item in vote_set_for_user:
            if item.create_time.date() == datetime.datetime.now().date():
                # check whether the date of the vote item equal to the date today
                num += 1
        return num
    
    
    def is_same_vote_item_today(sid, photographic_work_item_id):
        """
        check whether vote for the same item today
        """
        vote_set = VoteItem.objects.filter(school_id=sid, photographic_work_item_id=photographic_work_item_id)
        for item in vote_set:
            if item.create_time.date() == datetime.datetime.now().date():
                return True
        return False
    
    
