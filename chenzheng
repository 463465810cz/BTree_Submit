# include "utility.hpp"
# include <fstream>
# include <functional>
# include <cstddef>
# include "exception.hpp"

namespace sjtu {

    template <class Key, class Value, class Compare = std::less<Key> >
    class BTree {
    public:
        typedef pair <Key, Value> value_type;
        class iterator;
        class const_iterator;

    private:
        static const int M = 1000;
        static const int L = 200;
        static const int head_offset = 0;
        const char BtreeName[20]="Bpt.sjtu";


        struct FileHead
        {
            off_t leafhead; //叶节点的头部
            off_t leaftail; //叶节点的尾部
            off_t root; //B+树的根
            size_t allsize; //B+树的大小
            off_t FileEof; //所有文件的结尾位置//指向的是结束位置+1（即第一个空位置）
            FileHead(){leafhead=0;leaftail=0;root=0;allsize=0;FileEof=0;}
        }head;

        struct LeafNode
        {
            off_t offset; //叶节点的偏移量
            off_t Parent; //父节点
            off_t pre,next;//pre-前一个，next-后一个
            int curnum;//当前数量
            value_type data[L+10];//存放数据
            LeafNode()
            {
                offset=0;Parent=0;pre=0;next=0;curnum=0;
            }
        };

        struct InterNode
        {
            off_t offset; //叶节点的偏移量
            off_t Parent; //父节点
            off_t child[M+10]; //儿子节点
            Key key[M+10]; //键值
            int curnum;//当前key的数量(key-1/child-0)
            bool type; //0-后无叶子 1-后接叶子
            InterNode()
            {
                offset=0;Parent=0;
                for(int i=0;i<=M;++i)child[i]=0;
                curnum=0;type=0;
            }
        };

        static const size_t size_head= sizeof(FileHead);
        static const size_t size_node= sizeof(InterNode);
        static const size_t size_leaf= sizeof(LeafNode);

        FILE *fp;
        bool fp_open;
        bool file_already_exists;


        //文件操作
        inline void readFile(void *place, off_t  offset, size_t num, size_t size) const {
            if (fseek(fp, offset, SEEK_SET)) throw "open file failed!";
            fread(place, size, num, fp);
        }

        inline void writeFile(void *place, off_t  offset, size_t num, size_t size) const {
            if (fseek(fp, offset, SEEK_SET)) throw "open file failed!";
            fwrite(place, size, num, fp);
        }

        inline void build_tree()
        {
            file_already_exists=1;
            //开始偏移量为0，类似数组
            head.allsize=0;
            InterNode root;
            LeafNode leaf;
            head.root=root.offset=size_head;
            head.leafhead=head.leaftail=leaf.offset=head.root+size_node;
            head.FileEof=head.leafhead+size_leaf;
            root.Parent=0;root.curnum=0;root.type=1;root.child[0]=leaf.offset;
            leaf.Parent=root.offset;
            //存入数据
            writeFile(&head,head_offset,1,size_head);
            writeFile(&root,root.offset,1,size_node);
            writeFile(&leaf,leaf.offset,1,size_leaf);
        }


        void split_node(InterNode &node)
        {
            //根节点要单独考虑（把node.key[(node.curnum>>1)+1]推上去做key)

            if(node.offset == head.root)
            {
                //优先考虑根节点
                //更新根和自身
                InterNode new_root;
                InterNode NewNode;
                new_root.offset = head.FileEof;head.FileEof+=size_node;
                NewNode.offset = head.FileEof;head.FileEof += size_node;
                head.root = new_root.offset;node.Parent = head.root;NewNode.Parent = head.root;
                NewNode.curnum = node.curnum-(node.curnum>>1)-1;//分一个出去做新根
                for(int i = 1;i <= NewNode.curnum;i++)
                    NewNode.key[i] = node.key[i+(node.curnum>>1)+1];

                for(int i = 0;i <= NewNode.curnum;i++)
                    NewNode.child[i] = node.child[i+(node.curnum>>1)+1];

                new_root.curnum=1;new_root.Parent=0;new_root.type=0;NewNode.type=node.type;
                new_root.child[0]=node.offset;new_root.child[1]=NewNode.offset;new_root.key[1]=node.key[(node.curnum>>1)+1];
                node.curnum >>=1;

                //更新儿子节点
                if(node.type)
                {

                    LeafNode change;

                    for(int i = 0;i <= NewNode.curnum;i++){
                        readFile(&change,node.child[i+node.curnum+1],1,size_leaf);
                        change.Parent=NewNode.offset;
                        writeFile(&change,node.child[i+node.curnum+1],1,size_leaf);
                    }
                }
                else
                {
                    InterNode change;
                    for(int i = 0;i <= NewNode.curnum;i++)
                    {
                        readFile(&change,node.child[i+node.curnum+1],1,size_node);
                        change.Parent=NewNode.offset;
                        writeFile(&change,node.child[i+node.curnum+1],1,size_node);
                    }
                }

                //录入
                writeFile(&head,head_offset,1,size_head);
                writeFile(&node,node.offset,1,size_node);
                writeFile(&new_root,new_root.offset,1, size_node);
                writeFile(&NewNode,NewNode.offset,1, size_node);

            }
            else{

                //更新自己
                InterNode NewNode;
                NewNode.offset=head.FileEof;head.FileEof+=size_node;
                NewNode.curnum=node.curnum-(node.curnum>>1)-1;node.curnum>>=1;
                NewNode.Parent = node.Parent;NewNode.type = node.type;
                for (int i = 0;i <= NewNode.curnum;i++)
                    NewNode.child[i] = node.child[i+1+node.curnum];
                for(int i = 1;i <= NewNode.curnum;i++)
                    NewNode.key[i] = node.key[i+1+node.curnum];
                //录入
                writeFile(&node,node.offset,1, size_node);
                writeFile(&NewNode,NewNode.offset,1,size_node);
                writeFile(&head,head_offset,1,size_head);

                //更新儿子节点
                InterNode tmp;
                LeafNode leaf;
                for(int i = 0;i <= NewNode.curnum;i++)
                {
                    if(NewNode.type==1)
                    {
                        readFile(&leaf,NewNode.child[i],1,size_leaf);
                        leaf.Parent=NewNode.offset;
                        writeFile(&leaf,NewNode.child[i],1,size_leaf);
                    }
                    else
                    {
                        readFile(&tmp,NewNode.child[i],1,size_node);
                        tmp.Parent=NewNode.offset;
                        writeFile(&tmp,NewNode.child[i],1,size_node);
                    }
                }

                //更新父节点
                readFile(&tmp,node.Parent,1,size_node);
                insert_node(tmp,node.key[node.curnum+1],NewNode.offset);
            }
        }


        void insert_node(InterNode &node,const Key &key,const off_t child)
        {
            //插入时，空出child[0],分裂是在填满
            int pos = 1;
            while(pos<=node.curnum&&key>=node.key[pos])pos++;
            for(int i=node.curnum;i>=pos;i--)
            {
                node.key[i+1]=node.key[i];
                node.child[i+1]=node.child[i];
            }
            node.key[pos]=key;node.child[pos]=child;node.curnum++;
            if(node.curnum <= M)
                writeFile(&node,node.offset,1, size_node);
            else split_node(node);
        }

        void split_leaf(LeafNode &leaf,iterator &p,const Key &key)
        {
            LeafNode NewLeaf;
            NewLeaf.curnum=leaf.curnum>>1;leaf.curnum-=NewLeaf.curnum;
            NewLeaf.offset=head.FileEof;head.FileEof+=size_leaf;
            NewLeaf.Parent=leaf.Parent;NewLeaf.pre=leaf.offset;NewLeaf.next=leaf.next;leaf.next=NewLeaf.offset;
            if(NewLeaf.next==0){head.leaftail=NewLeaf.offset;writeFile(&head,head_offset,1,size_head);}
            else
            {
                LeafNode tmp;
                readFile(&tmp,NewLeaf.next,1, size_leaf);
                tmp.pre=NewLeaf.offset;
                writeFile(&tmp,tmp.offset,1, size_leaf);//覆盖
            }

            for(int i=0;i<NewLeaf.curnum;++i)
            {
                NewLeaf.data[i].first =leaf.data[i+leaf.curnum].first;
                NewLeaf.data[i].second=leaf.data[i+leaf.curnum].second;
                if(NewLeaf.data[i].first==key)
                {
                    p.offset=NewLeaf.offset;
                    p.place=i;
                    p.from=this;
                }
            }
            //写入
            writeFile(&leaf,leaf.offset,1,size_leaf);
            writeFile(&NewLeaf,NewLeaf.offset,1,size_leaf);

            //更新父节点(把NewLeaf.data[0].first推上去做索引）
            InterNode tmp;
            readFile(&tmp,leaf.Parent,1, size_node);
            insert_node(tmp,NewLeaf.data[0].first,NewLeaf.offset);

        }

        pair <iterator, OperationResult> insert_leaf( LeafNode &leaf,const Key &key,const Value &value)
        {
            iterator p;
            int pos=0;
            for(;pos<leaf.curnum;++pos)
            {
                iterator q;
                if(key==leaf.data[pos].first) return pair<iterator, OperationResult>(q,Fail);
                if(key<leaf.data[pos].first)break;
            }
            for(int i=leaf.curnum-1;i>=pos;--i)
            {
                leaf.data[i+1].first=leaf.data[i].first;
                leaf.data[i+1].second=leaf.data[i].second;
            }
            leaf.data[pos].first=key;leaf.data[pos].second=value;
            ++head.allsize;++leaf.curnum;
            p.offset=leaf.offset;p.place=pos;p.from=this;
            writeFile(&head,head_offset,1,size_head);
            if(leaf.curnum<=L)writeFile(&leaf,leaf.offset,1,size_leaf);
            else split_leaf(leaf,p,key);
            return pair<iterator, OperationResult>(p,Success);
        }

        //递归找叶子节点
        off_t locate_leaf(const Key &key,off_t offset)
        {
            InterNode tmp;
            readFile(&tmp,offset,1, size_node);
            if(tmp.type == 1)
            {
                int pos = 0;
                while(pos < tmp.curnum && key >= tmp.key[pos + 1])pos++;
                return tmp.child[pos];
            }
            else
            {
                int pos=0;
                while(pos < tmp.curnum && key >= tmp.key[pos + 1])pos++;
                return locate_leaf(key,tmp.child[pos]);
            }
        }

    public:
        class iterator {
            friend class BTree;
            friend class const_iterator;
        private:

            off_t offset;//指向的叶子节点的偏移量
            size_t place;//元素在指向的叶子结点中的位置
            BTree *from;
        public:

            bool modify(const Value& value) {
                LeafNode p;
                from-> readFile(&p, offset, 1, size_leaf);
                p.data[place].second = value;
                from-> writeFile(&p, offset, 1, size_leaf);
                return true;
            }

            iterator() {
                from = nullptr;place=0;offset=0;
            }

            iterator(const iterator& other) {
                from= other.from;
                offset = other.offset;
                place= other.place;
            }
            iterator(const const_iterator& other) {
                from= other.from;
                offset = other.offset;
                place= other.place;
            }

            // Return a new iterator which points to the n-next elements
            //后加
            iterator operator++(int) {
                iterator p=*this;
                if(*this==from->end())
                {
                    from= nullptr;place=0;offset=0;
                    return p;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==tmp.curnum-1)
                {
                    if(tmp.next==0)
                    {
                        place++;
                    }
                    else
                    {
                        offset=tmp.next;place=0;
                    }
                }
                else place++;
                return p;
            }
            //前加
            iterator& operator++() {
                if(*this==from->end())
                {
                    from= nullptr;place=0;offset=0;
                    return this;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==tmp.curnum-1)
                {
                    if(tmp.next==0)
                    {
                        place++;
                    }
                    else
                    {
                        offset=tmp.next;place=0;
                    }
                }
                else place++;
                return this;
            }
            //后减
            iterator operator--(int) {
                iterator p=*this;
                if(*this==from->begin())
                {
                    from= nullptr;place=0;offset=0;
                    return p;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==0)
                {
                    LeafNode T;
                    from->readFile(&T,tmp.pre,1,size_leaf);
                    place=T.curnum-1;offset=T.offset;
                }
                else place--;
                return p;
            }
            //前减
            iterator& operator--() {

                if(*this==from->begin())
                {
                    from= nullptr;place=0;offset=0;
                    return *this;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==0)
                {
                    LeafNode T;
                    from->readFile(&T,tmp.pre,1,size_leaf);
                    place=T.curnum-1;offset=T.offset;
                }
                else place--;
                return *this;
            }

            bool operator==(const iterator& rhs) const {
                return (offset==rhs.offset && place==rhs.place && from==rhs.from);
            }
            bool operator==(const const_iterator& rhs) const {
                return (offset==rhs.offset && place==rhs.place && from==rhs.from);
            }
            bool operator!=(const iterator& rhs) const {
                return (offset!=rhs.offset || place!=rhs.place || from!=rhs.from);
            }
            bool operator!=(const const_iterator& rhs) const {
                return (offset!=rhs.offset || place!=rhs.place || from!=rhs.from);
            }
            Key getvalue(){
                LeafNode leaf;
                from->readFile(&leaf,offset,1, size_leaf);
                return leaf.data[place].second;
            }
        };

        class const_iterator {
            friend class BTree;
            friend class iterator;
        private:

            off_t offset;//指向的叶子节点的偏移量
            size_t place;//元素在指向的叶子结点中的位置
            BTree *from;

        public:
            const_iterator() {
                from = nullptr;place=0;offset=0;
            }

            const_iterator(const iterator& other) {
                from= other.from;
                offset = other.offset;
                place= other.place;
            }
            const_iterator(const const_iterator& other) {
                from= other.from;
                offset = other.offset;
                place= other.place;
            }

            // Return a new iterator which points to the n-next elements
            //后加
            const_iterator operator++(int) {
                const_iterator p=*this;
                if(*this==from->end())
                {
                    from= nullptr;place=0;offset=0;
                    return p;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==tmp.curnum-1)
                {
                    if(tmp.next==0)
                    {
                        place++;
                    }
                    else
                    {
                        offset=tmp.next;place=0;
                    }
                }
                else place++;
                return p;
            }
            //前加
            const_iterator& operator++() {
                if(*this==from->end())
                {
                    from= nullptr;place=0;offset=0;
                    return this;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==tmp.curnum-1)
                {
                    if(tmp.next==0)
                    {
                        place++;
                    }
                    else
                    {
                        offset=tmp.next;place=0;
                    }
                }
                else place++;
                return this;
            }
            //后减
            const_iterator operator--(int) {
                const_iterator p=*this;
                if(*this==from->begin())
                {
                    from= nullptr;place=0;offset=0;
                    return p;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==0)
                {
                    LeafNode T;
                    from->readFile(&T,tmp.pre,1,size_leaf);
                    place=T.curnum-1;offset=T.offset;
                }
                else place--;
                return p;
            }
            //前减
            const_iterator& operator--() {

                if(*this==from->begin())
                {
                    from= nullptr;place=0;offset=0;
                    return *this;
                }
                LeafNode tmp;
                from->readFile(&tmp,offset,1,size_leaf);
                if(place==0)
                {
                    LeafNode T;
                    from->readFile(&T,tmp.pre,1,size_leaf);
                    place=T.curnum-1;offset=T.offset;
                }
                else place--;
                return *this;
            }

            bool operator==(const iterator& rhs) const {
                return (offset==rhs.offset && place==rhs.place && from==rhs.from);
            }
            bool operator==(const const_iterator& rhs) const {
                return (offset==rhs.offset && place==rhs.place && from==rhs.from);
            }
            bool operator!=(const iterator& rhs) const {
                return (offset!=rhs.offset || place!=rhs.place || from!=rhs.from);
            }
            bool operator!=(const const_iterator& rhs) const {
                return (offset!=rhs.offset || place!=rhs.place || from!=rhs.from);
            }
        };

        // Default Constructor and Copy Constructor

        BTree() {

            file_already_exists = 1;
            if (fp_open == 0) {
                fp = fopen(BtreeName, "rb+");
                if (fp == nullptr) {
                    file_already_exists = 0;
                    fp = fopen(BtreeName, "w");
                    fclose(fp);
                    fp = fopen(BtreeName, "rb+");
                } else readFile(&head, head_offset, 1, size_head);
                fp_open = 1;
            }
            if (file_already_exists == 0) build_tree();
        }


        ~BTree() {
            if (fp_open == 1) {
                fclose(fp);
                fp_open = 0;
            }
        }

        /**
         * Insert: Insert certain Key-Value into the database
         * Return a pair, the first of the pair is the iterator point to the new
         * element, the second of the pair is Success if it is successfully inserted
         */
        pair <iterator, OperationResult> insert(const Key& key, const Value& value)
        {
            off_t  leaf_offset = locate_leaf(key, head.root);
            LeafNode leaf;
            readFile(&leaf,leaf_offset,1,size_leaf);
            pair <iterator,OperationResult> ret=insert_leaf(leaf,key,value);
            return ret;
        }




        // Erase: Erase the Key-Value
        // Return Success if it is successfully erased
        // Return Fail if the key doesn't exist in the database
        OperationResult erase(const Key& key) {
            // TODO erase function
            return Fail;  // If you can't finish erase part, just remaining here.
        }





        // Return a iterator to the beginning


        iterator begin()
        {
            iterator p;
            p.from=this;p.offset=head.leafhead;p.place=0;
            return p;
        }
        const_iterator cbegin() const
        {
            const_iterator p;
            p.from=this;p.offset=head.leafhead;p.place=0;
            return p;
        }
        // Return a iterator to the end(the next element after the last)
        iterator end()
        {
            iterator p;
            LeafNode tmp;
            readFile(&tmp,head.leaftail,1,size_leaf);
            p.from=this;p.offset=head.leaftail;p.place=tmp.curnum;
            return p;
        }
        const_iterator cend() const
        {
            const_iterator p;
            LeafNode tmp;
            readFile(&tmp,head.leaftail,1,size_leaf);
            p.from=this;p.offset=head.leaftail;p.place=tmp.curnum;
            return p;
        }

        // Check whether this BTree is empty
        bool empty() const { return head.allsize==0;}
        // Return the number of <K,V> pairs
        size_t size() const { return head.allsize;}
        // Clear the BTree

        void clear() {
            fp = fopen(BtreeName, "w");
            fclose(fp);
            fp = fopen(BtreeName, "rb+");
            build_tree();
        }

        Value at(const Key& key)
        {
            iterator p;
            LeafNode leaf;
            p=find(key);
            if(p==end()){throw "No Find";}
            readFile(&leaf,p.offset,1,size_leaf);
            return leaf.data[p.pos].second;
        }
        /**
         * Returns the number of elements with key
         *   that compares equivalent to the specified argument,
         * The default method of check the equivalence is !(a < b || b > a)
         */
        size_t count(const Key& key) const
        {
            if(find(key)==end())return 0;
            else return 1;
        }
        /**
         * Finds an element with key equivalent to key.
         * key value of the element to search for.
         * Iterator to an element with key equivalent to key.
         *   If no such element is found, past-the-end (see end()) iterator is
         * returned.
         */
        iterator find(const Key& key)
        {
            off_t offset=locate_leaf(key,head.root);
            LeafNode leaf;
            iterator p;
            readFile(&leaf,offset,1,size_leaf);
            for(int i=0;i<leaf.curnum;++i)
            {
                if(leaf.data[i].first==key)
                {
                    p.place=i;p.from=this;p.offset=offset;
                    return p;
                }
            }
            return end();
        }
        const_iterator find(const Key& key) const
        {
            off_t offset=locate_leaf(key,head.root);
            LeafNode leaf;
            const_iterator p;
            readFile(&leaf,offset,1,size_leaf);
            for(int i=0;i<leaf.curnum;++i)
            {
                if(leaf.data[i].first==key)
                {
                    p.place=i;p.from=this;p.offset=offset;
                    return p;
                }
            }
            return cend();
        }

    };
}  // namespace sjtu
